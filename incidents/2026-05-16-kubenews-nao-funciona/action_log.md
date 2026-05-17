# Log de Ações — 2026-05-16 — kubenews-nao-funciona

## Status final
RESOLVIDO — 3 iterações

## Estado final dos pods
```
NAME                         READY   STATUS    RESTARTS   AGE     IP             NODE
kube-news-7649dfcf6b-l9lc5   1/1     Running   0          18s     10.109.0.123   pool-g1669fym6-33r5ds
postgres-8647476c7b-tb5gh    1/1     Running   0          9m18s   10.109.0.181   pool-g1669fym6-33r5dp
```

## Ações executadas

### Iteração 1 — 2026-05-16T22:00:00Z
**Hipótese:** O volume block storage (ext4) cria `lost+found` na raiz, impedindo o `initdb` do PostgreSQL de inicializar o diretório `/var/lib/postgresql/data` (que não pode estar não-vazio). Isso derruba o postgres em CrashLoopBackOff, e o kube-news falha em cascata com ECONNREFUSED.

**Ação:** Editado `/home/fabricioveronez/imersao/kube-news/k8s-bo/postgres-deployment.yml` adicionando `subPath: pgdata` ao volumeMount do container postgres, fazendo o initdb usar o subdiretório `/var/lib/postgresql/data/pgdata` (livre de `lost+found`). Manifesto aplicado via `kubectl apply`.

**Resultado:** Kubernetes iniciou um rolling update criando novo pod `postgres-8647476c7b-tb5gh` no node `pool-g1669fym6-33r5dp`. Porém o pod ficou preso em `ContainerCreating` com erro `Multi-Attach error` — o PVC (RWO/block storage) ainda estava anexado ao pod antigo `postgres-85477b5999-scg4b` no node `pool-g1669fym6-33r5ds`.

**Estado dos pods após:** postgres antigo: CrashLoopBackOff | postgres novo: ContainerCreating | kube-news: CrashLoopBackOff

---

### Iteração 2 — 2026-05-16T22:03:00Z
**Hipótese:** O volume RWO está bloqueado pelo pod antigo do ReplicaSet anterior, que o Kubernetes não remove durante o rolling update enquanto o novo pod não ficar pronto — deadlock. É preciso remover o pod antigo e escalar o ReplicaSet antigo para 0 manualmente.

**Ação:**
1. Deletado com force o pod `postgres-85477b5999-scg4b` (pod do ReplicaSet antigo que segurava o PVC).
2. O ReplicaSet antigo recriou um novo pod — deletado também com force (`postgres-85477b5999-c94jj`).
3. Escalado o ReplicaSet antigo `postgres-85477b5999` para 0 réplicas para impedir recriação.

**Resultado:** Com o ReplicaSet antigo em 0 e nenhum pod concorrendo pelo PVC, o volume foi desanexado do node antigo. O pod `postgres-8647476c7b-tb5gh` conseguiu anexar o volume no node `pool-g1669fym6-33r5dp` e transitou para `Running`.

**Estado dos pods após:** postgres novo: Running (1/1) | kube-news: CrashLoopBackOff (em back-off, aguardando próximo restart)

---

### Iteração 3 — 2026-05-16T22:12:00Z
**Hipótese:** O kube-news está em back-off e não tentará reconectar automaticamente em tempo hábil — o último crash ocorreu antes do postgres subir. Deletar o pod força um restart imediato.

**Ação:** Deletado o pod `kube-news-7649dfcf6b-ct79r`. O Deployment recriou imediatamente um novo pod (`kube-news-7649dfcf6b-l9lc5`) que conectou com sucesso ao postgres já disponível.

**Resultado:** kube-news conectou ao banco e transitou para Running (1/1). Aplicação disponível no LoadBalancer `134.209.130.140:80`.

**Estado dos pods após:** postgres: 1/1 Running | kube-news: 1/1 Running — TODOS OS PODS HEALTHY

---

## Lições aprendidas

1. **subPath é obrigatório com block storage ext4:** Volumes DigitalOcean `do-block-storage` formatam o disco com ext4, criando `lost+found` na raiz. Usar `subPath` no volumeMount do postgres é sempre necessário nesse ambiente.

2. **Rolling update com PVC RWO pode causar deadlock:** Um Deployment com estratégia `RollingUpdate` e PVC `ReadWriteOnce` pode entrar em deadlock se o novo pod for agendado em um node diferente do pod antigo — o PVC não pode ser reanexado enquanto o pod antigo estiver rodando (mesmo em crash). Mitigação: usar `strategy: type: Recreate` em Deployments com PVC RWO.

3. **Ordem de aplicação dos manifestos importa:** O Secret deve ser aplicado antes dos Deployments para evitar o erro `secret not found` nos primeiros eventos do pod kube-news.

# Pós-Mortem: Falha total da aplicação kube-news no namespace default

**Data do incidente:** 2026-05-16  
**Duração:** ~15 minutos (21:30 – 21:45 UTC)  
**Severidade:** Crítica — aplicação completamente indisponível  
**Status:** Resolvido  
**Autor:** Fabricio Veronez  

---

## Resumo executivo

Após o deploy inicial da aplicação kube-news no cluster `do-nyc1-k8s-aula` (DigitalOcean, região nyc1), ambos os pods (`postgres` e `kube-news`) entraram em `CrashLoopBackOff` imediatamente, tornando a aplicação totalmente indisponível. A causa raiz foi o volume PVC provisionado pelo DigitalOcean Block Storage montado diretamente na raiz do diretório de dados do PostgreSQL, que continha um diretório `lost+found` impedindo a inicialização do banco.

---

## Linha do tempo

| Horário (UTC) | Evento |
|---|---|
| 21:30:02 | Deploy aplicado no cluster — pods `postgres` e `kube-news` criados |
| 21:30:03 | Primeiro erro registrado: `secret "postgres-secret" not found` no pod kube-news (race condition) |
| 21:30:04 | Pod `postgres` inicia — `initdb` falha com `lost+found directory` no volume |
| 21:30:42 | Pod `kube-news` crasheia com `ECONNREFUSED 10.108.46.32:5432` — postgres nunca subiu |
| 21:30:52 | Ambos os pods em `CrashLoopBackOff` confirmado |
| 21:33:xx | Início da investigação via MCP Kubernetes |
| 21:34:xx | Causa raiz identificada — patch aplicado no Deployment postgres com `subPath` e `PGDATA` |
| 21:35:xx | Descoberto problema secundário: `Permission denied` ao criar subdiretório no volume |
| 21:36:xx | `initContainer` `fix-permissions` adicionado para corrigir ownership do diretório |
| 21:37:xx | Descoberto que diretório `pgdata` criado por tentativas anteriores bloqueava `initdb` |
| 21:37:xx | `initContainer` atualizado para apagar e recriar `pgdata` antes de iniciar |
| 21:40:19 | Pod `postgres-58d5f7ddf-pp7zk` atinge status `1/1 Running` |
| 21:40:32 | `rollout restart` aplicado no Deployment `kube-news` |
| 21:40:47 | Pod `kube-news-6995747746-gtr22` atinge status `1/1 Running` |
| 21:40:47 | **Incidente encerrado — aplicação totalmente operacional** |

---

## Causa raiz

### Problema principal: `lost+found` no volume Block Storage

O PVC `postgres-pvc` foi provisionado pelo DigitalOcean Block Storage (`do-block-storage`). Ao ser formatado, esse tipo de volume cria automaticamente um diretório `lost+found` na raiz do filesystem. O Deployment do PostgreSQL montava esse volume diretamente em `/var/lib/postgresql/data` sem nenhuma configuração adicional, fazendo com que o `initdb` encontrasse o diretório não vazio e recusasse a inicialização.

**Evidência — log do pod `postgres-85477b5999-q2vlk`:**
```
initdb: error: directory "/var/lib/postgresql/data" exists but is not empty
initdb: detail: It contains a lost+found directory, perhaps due to it being a mount point.
initdb: hint: Using a mount point directly as the data directory is not recommended.
            Create a subdirectory under the mount point.
```

### Problema secundário (cascata): kube-news sem banco disponível

Com o PostgreSQL nunca inicializando, o `kube-news` tentou conectar na porta 5432 e recebeu `ECONNREFUSED`, crasheando também.

**Evidência — log do pod `kube-news-7649dfcf6b-7fq2q`:**
```
ConnectionRefusedError [SequelizeConnectionRefusedError]: connect ECONNREFUSED 10.108.46.32:5432
```

### Problema terciário (race condition histórica): Secret ausente

No primeiro ciclo do pod `kube-news`, o Secret `postgres-secret` ainda não havia sido criado (problema de ordem de aplicação dos manifests). Resolvido automaticamente após re-criação do pod, mas indica fragilidade no processo de deploy.

**Evidência — evento do pod `kube-news-7649dfcf6b-7fq2q`:**
```
Warning  Failed  84s  kubelet  spec.containers{kube-news}: Error: secret "postgres-secret" not found
```

---

## Impacto

- **Disponibilidade:** 0% durante o período do incidente
- **Usuários afetados:** Todos — aplicação completamente inacessível via NodePort 30000
- **Dados:** Nenhuma perda de dados (banco nunca chegou a inicializar)
- **Serviços dependentes:** Nenhum identificado neste ambiente

---

## Resolução aplicada

### Alteração 1: variável `PGDATA` e `initContainer` de permissões

Adicionados ao Deployment `postgres`:

```yaml
spec:
  template:
    spec:
      initContainers:
        - name: fix-permissions
          image: busybox
          command:
            - sh
            - -c
            - rm -rf /var/lib/postgresql/data/pgdata && mkdir -p /var/lib/postgresql/data/pgdata && chown -R 999:999 /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
      containers:
        - name: postgres
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          # demais envs via Secret...
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
```

**Por que funciona:** A variável `PGDATA` direciona o PostgreSQL a usar o subdiretório `pgdata` em vez da raiz do volume, evitando o conflito com o `lost+found`. O `initContainer` garante que o diretório exista com o owner correto (UID 999 = usuário `postgres` na imagem Alpine) antes do container principal iniciar.

### Alteração 2: restart do kube-news

Após o postgres estabilizar:
```bash
kubectl rollout restart deployment/kube-news -n default
```

---

## Manifesto correto (referência)

O arquivo `k8s/kube-news.yml` no repositório já continha a configuração correta com `subPath: pgdata` e `PGDATA`. O incidente ocorreu porque os manifests aplicados no cluster divergiam do repositório. **Os manifests do cluster devem sempre ser derivados do repositório.**

---

## Lições aprendidas

### O que deu errado

1. **Manifests aplicados no cluster divergiam do repositório** — o arquivo `k8s/kube-news.yml` já tinha `subPath: pgdata` e `PGDATA`, mas o que foi aplicado no namespace `default` não tinha essas configurações.
2. **Sem `PGDATA` configurado, PostgreSQL + Block Storage = falha garantida** — qualquer volume provisionado por CSI drivers que criam `lost+found` (ext4) causará o mesmo problema.
3. **Sem `initContainer` de permissões, a criação do subdiretório falha** — o diretório `pgdata` precisa existir com owner correto antes do postgres iniciar.
4. **PVC RWO em cluster multi-nó causa `Multi-Attach` durante rollout** — ao trocar de nó, o volume bloqueia até o pod anterior ser deletado completamente.

### O que funcionou bem

1. **Readiness e Liveness probes** configurados corretamente impediram que tráfego chegasse a pods não saudáveis.
2. **Secret separado** (`postgres-secret`) manteve credenciais fora dos manifests principais.
3. **Diagnóstico via logs e eventos** foi suficiente para identificar a causa raiz sem acesso direto aos nós.

---

## Ações preventivas

| # | Ação | Responsável | Prazo |
|---|------|-------------|-------|
| 1 | Atualizar `k8s/kube-news.yml` no namespace `default` para refletir o estado corrigido (com `initContainer` e `PGDATA`) | Time de infra | Imediato |
| 2 | Adicionar ao runbook de deploy: sempre aplicar manifests a partir do repositório, nunca manualmente | Time de infra | Imediato |
| 3 | Incluir `initContainer` de permissões como padrão em todos os Deployments com PVC Block Storage | Time de infra | Próximo sprint |
| 4 | Adicionar `initContainer` de wait-for-db no `kube-news` para evitar crash em race condition com o postgres | Time de dev | Próximo sprint |
| 5 | Implementar pipeline CI/CD com `kubectl diff` para detectar divergência entre repositório e cluster antes de aplicar | Time de infra | Backlog |

---

## Guia de resolução para incidentes futuros

### Sintomas que indicam este problema

- Pods do PostgreSQL em `CrashLoopBackOff` logo após o primeiro deploy
- Log contendo: `initdb: error: directory ... exists but is not empty`
- Log contendo: `lost+found directory`
- Pod da aplicação com `ECONNREFUSED` na porta 5432

### Passos de diagnóstico

```bash
# 1. Verificar status dos pods
kubectl get pods -n <namespace> -o wide

# 2. Ver logs do postgres
kubectl logs <pod-postgres> -n <namespace>

# 3. Ver eventos do pod (Multi-Attach, Secret not found, etc.)
kubectl describe pod <pod-postgres> -n <namespace>
```

### Passos de correção

```bash
# 1. Patch no Deployment postgres — adicionar PGDATA e initContainer
kubectl patch deployment postgres -n <namespace> --type=strategic -p '{
  "spec": {
    "template": {
      "spec": {
        "initContainers": [{
          "name": "fix-permissions",
          "image": "busybox",
          "command": ["sh", "-c", "rm -rf /var/lib/postgresql/data/pgdata && mkdir -p /var/lib/postgresql/data/pgdata && chown -R 999:999 /var/lib/postgresql/data/pgdata"],
          "volumeMounts": [{"name": "postgres-data", "mountPath": "/var/lib/postgresql/data"}]
        }],
        "containers": [{
          "name": "postgres",
          "env": [{"name": "PGDATA", "value": "/var/lib/postgresql/data/pgdata"}]
        }]
      }
    }
  }
}'

# 2. Se o novo pod travar em ContainerCreating com Multi-Attach,
#    deletar o pod antigo do ReplicaSet anterior
kubectl delete pod <pod-antigo> -n <namespace> --grace-period=0

# 3. Após postgres estar 1/1 Running, reiniciar a aplicação
kubectl rollout restart deployment/<app> -n <namespace>

# 4. Verificar que tudo está Running
kubectl get pods -n <namespace>
```

### Armadilhas conhecidas durante a correção

- **Multi-Attach (RWO):** PVCs `ReadWriteOnce` só podem ser anexados a um nó por vez. Durante rollouts que trocam de nó, o pod antigo precisa ser deletado com `--grace-period=0` para liberar o volume rapidamente.
- **ReplicaSets antigos recriam pods:** Deletar apenas o pod não é suficiente se o ReplicaSet antigo ainda existir. Delete o ReplicaSet antigo após o novo estar no ar.
- **Diretório `pgdata` parcialmente criado:** Se tentativas anteriores criaram o diretório `pgdata` vazio, o `initdb` recusará mesmo sem `lost+found`. O `initContainer` deve usar `rm -rf` antes de `mkdir`.

---

## Referências

- [PostgreSQL: Using a mount point directly as the data directory](https://www.postgresql.org/docs/current/creating-cluster.html)
- [DigitalOcean Block Storage — ext4 e lost+found](https://docs.digitalocean.com/products/volumes/)
- [Kubernetes: Persistent Volumes — Access Modes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)
- Manifesto de referência: `k8s/kube-news.yml` (raiz do repositório)

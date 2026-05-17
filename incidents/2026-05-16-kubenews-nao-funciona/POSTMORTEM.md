# Pós-Mortem: PostgreSQL em CrashLoopBackOff por conflito de volume ext4 derruba toda a stack Kubenews

**Data do incidente:** 2026-05-16
**Duração:** 22:00Z até 22:12Z (aproximadamente 12 minutos)
**Severidade:** Crítica
**Status:** Resolvido
**Namespace:** default

---

## Resumo executivo

O PostgreSQL entrou em `CrashLoopBackOff` imediatamente após o deploy porque o volume block storage (DigitalOcean `do-block-storage`, formatado em ext4) cria automaticamente um diretório `lost+found` na raiz, e o `initdb` do PostgreSQL recusa inicializar qualquer diretório não-vazio. Como o postgres nunca subiu, o service não tinha endpoints ativos e a aplicação Node.js (`kube-news`) falhou em cascata com `ECONNREFUSED`, tornando a aplicação 100% indisponível. A resolução exigiu três iterações: correção do manifesto com `subPath: pgdata`, remoção manual do pod antigo que segurava o PVC em deadlock durante o rolling update, e restart forçado do pod `kube-news`.

---

## Linha do tempo

| Horário | Evento |
|---|---|
| 2026-05-16T22:00:00Z | Investigação iniciada — ambos os pods em `CrashLoopBackOff` |
| 2026-05-16T22:00:00Z | Causa raiz identificada: `initdb` falha por `lost+found` no volume ext4 |
| 2026-05-16T22:00:00Z | **Iteração 1:** Manifesto `postgres-deployment.yml` corrigido com `subPath: pgdata` e aplicado via `kubectl apply` |
| 2026-05-16T22:00:00Z | Kubernetes inicia rolling update — novo pod `postgres-8647476c7b-tb5gh` criado, mas fica preso em `ContainerCreating` |
| 2026-05-16T22:00:00Z | Identificado `Multi-Attach error`: PVC RWO ainda anexado ao pod antigo no node diferente |
| 2026-05-16T22:03:00Z | **Iteração 2:** Pod antigo `postgres-85477b5999-scg4b` deletado com force |
| 2026-05-16T22:03:00Z | ReplicaSet antigo recria pod `postgres-85477b5999-c94jj` — deletado com force também |
| 2026-05-16T22:03:00Z | ReplicaSet antigo `postgres-85477b5999` escalado para 0 réplicas manualmente |
| 2026-05-16T22:03:00Z | PVC desanexado do node antigo — pod `postgres-8647476c7b-tb5gh` transita para `Running (1/1)` |
| 2026-05-16T22:12:00Z | **Iteração 3:** Pod `kube-news-7649dfcf6b-ct79r` deletado para forçar restart imediato |
| 2026-05-16T22:12:00Z | Novo pod `kube-news-7649dfcf6b-l9lc5` criado — conecta ao postgres com sucesso |
| 2026-05-16T22:12:00Z | Ambos os pods em `Running (1/1)` — aplicação disponível em `134.209.130.140:80` |

---

## Causa raiz

O manifesto `k8s-bo/postgres-deployment.yml` montava o volume diretamente em `/var/lib/postgresql/data` sem `subPath`:

```yaml
volumeMounts:
  - name: postgres-data
    mountPath: /var/lib/postgresql/data
```

O volume `do-block-storage` da DigitalOcean é formatado com ext4, que cria automaticamente o diretório `lost+found` na raiz do filesystem ao formatar. O `initdb` do PostgreSQL recusa inicializar qualquer diretório que **não esteja completamente vazio**, abortando com exit code 1 a cada tentativa:

```
initdb: error: directory "/var/lib/postgresql/data" exists but is not empty
initdb: detail: It contains a lost+found directory, perhaps due to it being a mount point.
initdb: hint: Using a mount point directly as the data directory is not recommended.
                Create a subdirectory under the mount point.
```

Evidência nos logs do pod `postgres-85477b5999-scg4b`:

```
The files belonging to this database system will be owned by user "postgres".
...
initdb: error: directory "/var/lib/postgresql/data" exists but is not empty
initdb: detail: It contains a lost+found directory, perhaps due to it being a mount point.
initdb: hint: Using a mount point directly as the data directory is not recommended.
```

O postgres nunca inicializou o cluster de banco de dados. Cada tentativa terminava com exit code 1, entrando em `CrashLoopBackOff`.

### Causas secundárias (cascata)

**1. kube-news sem banco disponível (ECONNREFUSED)**

Como o pod postgres nunca saiu do CrashLoopBackOff, o service `postgres` (ClusterIP `10.108.49.107:5432`) não tinha endpoints ativos. A aplicação Node.js subia na porta 8080 mas falhava imediatamente ao tentar conectar via Sequelize:

```
Aplicação rodando na porta 8080
ConnectionRefusedError [SequelizeConnectionRefusedError]: connect ECONNREFUSED 10.108.49.107:5432
    errno: -111,
    code: 'ECONNREFUSED',
    address: 10.108.49.107,
    port: 5432
```

Isso gerou o segundo `CrashLoopBackOff`, tornando toda a aplicação indisponível.

**2. Multi-Attach deadlock durante rolling update (RWO + nós diferentes)**

Ao aplicar o manifesto corrigido com `kubectl apply`, o Kubernetes criou um novo pod em um node diferente do pod antigo. Como o PVC é `ReadWriteOnce` (RWO), o volume só pode ser anexado a um node por vez. O pod antigo, mesmo em CrashLoopBackOff, ainda segurava o PVC anexado. O novo pod ficou preso em `ContainerCreating` com:

```
Multi-Attach error for volume "pvc-cc422e10...": volume is already exclusively attached to one node
```

O rolling update entrou em deadlock: o Kubernetes não remove o pod antigo até o novo ficar pronto, mas o novo não fica pronto porque o volume está preso no antigo.

**3. Ordem de aplicação dos manifestos (Secret não encontrado)**

Nos primeiros eventos do pod `kube-news`, os eventos registraram:

```
Warning  Failed   89s (x2 over 90s)   kubelet  spec.containers{kube-news}: Error: secret "postgres-secret" not found
```

Os Deployments foram aplicados antes do Secret `postgres-secret` existir no cluster. O Secret foi criado posteriormente e o erro não se repetiu, mas indica problema de ordem de aplicação dos manifestos.

**Cadeia de falhas:**
```
block storage ext4 cria lost+found
  → initdb recusa inicializar /var/lib/postgresql/data
    → postgres pod: CrashLoopBackOff (exit 1)
      → service postgres sem endpoints ativos
        → kube-news: ECONNREFUSED ao tentar conectar ao DB
          → kube-news pod: CrashLoopBackOff (exit 1)
            → aplicação 100% indisponível para usuários
```

---

## Impacto

- **Disponibilidade:** 100% de indisponibilidade — nenhum usuário conseguia acessar a aplicação durante todo o período do incidente
- **Pods afetados:** `kube-news-7649dfcf6b-ct79r` (CrashLoopBackOff), `postgres-85477b5999-scg4b` (CrashLoopBackOff)
- **Dados:** Nenhuma perda de dados — o PostgreSQL nunca inicializou, portanto não havia dados a perder. O volume PVC permaneceu íntegro durante toda a operação de correção

---

## Resolução aplicada

### Passo 1 — Correção do manifesto `postgres-deployment.yml`

Adicionado `subPath: pgdata` ao `volumeMount` do container postgres:

```yaml
# k8s-bo/postgres-deployment.yml — ANTES
volumeMounts:
  - name: postgres-data
    mountPath: /var/lib/postgresql/data

# k8s-bo/postgres-deployment.yml — DEPOIS
volumeMounts:
  - name: postgres-data
    mountPath: /var/lib/postgresql/data
    subPath: pgdata
```

O `subPath` faz o postgres usar o subdiretório `/var/lib/postgresql/data/pgdata` dentro do volume montado. Esse subdiretório é criado pelo Kubernetes e está vazio, sem `lost+found`, permitindo que o `initdb` inicialize com sucesso.

```bash
kubectl apply -f /home/fabricioveronez/imersao/kube-news/k8s-bo/postgres-deployment.yml
```

### Passo 2 — Resolução do deadlock de PVC RWO

O rolling update entrou em deadlock com `Multi-Attach error`. Foi necessário liberar o PVC manualmente:

```bash
# 1. Forçar a remoção do pod antigo que segurava o PVC
kubectl delete pod postgres-85477b5999-scg4b --force --grace-period=0 -n default

# 2. O ReplicaSet antigo recriou um novo pod — deletar também
kubectl delete pod postgres-85477b5999-c94jj --force --grace-period=0 -n default

# 3. Escalar o ReplicaSet antigo para 0 para impedir novas recriações
kubectl scale replicaset postgres-85477b5999 --replicas=0 -n default
```

Com isso, o PVC foi desanexado do node antigo e o novo pod `postgres-8647476c7b-tb5gh` conseguiu anexar o volume e inicializar com sucesso.

### Passo 3 — Forçar restart do kube-news

O pod `kube-news` estava em back-off exponencial e não tentaria reconectar cedo. Com o postgres já disponível, foi forçado um restart imediato:

```bash
kubectl delete pod kube-news-7649dfcf6b-ct79r -n default
```

O Deployment recriou o pod `kube-news-7649dfcf6b-l9lc5`, que conectou ao postgres imediatamente e transitou para `Running (1/1)`.

### Verificação final

```bash
kubectl get pods -n default
# NAME                         READY   STATUS    RESTARTS   AGE
# kube-news-7649dfcf6b-l9lc5   1/1     Running   0          18s
# postgres-8647476c7b-tb5gh    1/1     Running   0          9m18s
```

Aplicação disponível em `http://134.209.130.140:80`.

---

## Lições aprendidas

### O que deu errado

1. **Manifesto sem `subPath` em ambiente com block storage ext4:** O manifesto `postgres-deployment.yml` foi criado sem considerar o comportamento do `do-block-storage` da DigitalOcean, que formata o volume com ext4 e sempre cria `lost+found` na raiz. Isso é um erro de configuração conhecido e documentado no próprio `initdb`.

2. **Estratégia `RollingUpdate` incompatível com PVC RWO em múltiplos nodes:** O Deployment usava a estratégia padrão `RollingUpdate`, que cria o novo pod antes de remover o antigo. Com um PVC `ReadWriteOnce`, isso resulta em deadlock quando os pods são agendados em nodes diferentes — situação que ocorreu neste incidente.

3. **Ordem de aplicação dos manifestos não documentada:** Os Deployments foram aplicados antes do Secret existir, causando erros iniciais `secret not found`. Não havia um runbook ou script que enforçasse a ordem correta.

### O que funcionou bem

1. **Diagnóstico rápido:** A combinação de `kubectl describe pod` e `kubectl logs` revelou a causa raiz (erro do `initdb`) em menos de 5 minutos.

2. **O volume permaneceu íntegro:** Como o postgres nunca chegou a inicializar, não havia dados no volume. A operação de corrigir o manifesto e recriar o pod foi de baixo risco.

3. **O Secret estava correto:** O conteúdo do `postgres-secret` estava alinhado com o manifesto em repositório, eliminando uma possível classe de erros de configuração de credenciais.

---

## Ações preventivas

| # | Ação | Prazo |
|---|---|---|
| 1 | Adicionar `subPath: pgdata` ao manifesto `k8s-bo/postgres-deployment.yml` (já aplicado em produção, mas confirmar no repositório com commit) | Imediato |
| 2 | Alterar a estratégia do Deployment do postgres para `type: Recreate` para evitar deadlock de PVC RWO em rolling updates futuros | Imediato |
| 3 | Criar script de deploy `deploy.sh` (ou `kustomization.yaml`) que aplica os manifestos na ordem correta: secret → pvc → services → deployments | Curto prazo |
| 4 | Adicionar `readinessProbe` no container `kube-news` para que o pod não receba tráfego antes de conectar ao banco | Curto prazo |
| 5 | Adicionar `initContainer` no Deployment do `kube-news` que aguarda o postgres estar disponível antes de iniciar o container principal | Médio prazo |
| 6 | Documentar em `CLAUDE.md` ou `README.md` do projeto que volumes `do-block-storage` sempre requerem `subPath` em montagens PostgreSQL | Curto prazo |

---

## Guia para incidentes futuros

### Sintomas que indicam este problema

- Pods `postgres-*` em `CrashLoopBackOff` logo após o deploy
- Pods `kube-news-*` em `CrashLoopBackOff` com logs mostrando `ECONNREFUSED` para o IP do service postgres
- `kubectl logs` do postgres mostrando mensagem sobre `lost+found`
- `kubectl describe pod` do postgres com eventos de `FailedScheduling` por PVC não encontrado seguidos de `CrashLoopBackOff`

### Passos de diagnóstico

```bash
# 1. Verificar estado geral dos pods
kubectl get pods -n default -o wide

# 2. Verificar logs do postgres para identificar o erro de initdb
kubectl logs <nome-do-pod-postgres> -n default
# Procurar por: "initdb: error: directory ... exists but is not empty"

# 3. Verificar eventos do pod postgres para identificar problemas de scheduling/PVC
kubectl describe pod <nome-do-pod-postgres> -n default
# Procurar por: "FailedScheduling", "Multi-Attach error", "lost+found"

# 4. Verificar logs do kube-news para confirmar a causa em cascata
kubectl logs <nome-do-pod-kube-news> -n default
# Procurar por: "ECONNREFUSED", "SequelizeConnectionRefusedError"

# 5. Verificar se o Secret existe e tem as chaves corretas
kubectl get secret postgres-secret -n default -o yaml
# Deve conter: db-database, db-username, db-password

# 6. Verificar o PVC
kubectl get pvc -n default
# Deve estar: Bound

# 7. Verificar o volumeMount no manifesto do postgres
kubectl get deployment postgres -n default -o yaml | grep -A5 volumeMounts
# Verificar se subPath: pgdata está presente
```

### Passos de correção

```bash
# PASSO 1: Corrigir o manifesto (se subPath estiver ausente)
# Editar k8s-bo/postgres-deployment.yml e adicionar subPath: pgdata
# Depois aplicar:
kubectl apply -f /home/fabricioveronez/imersao/kube-news/k8s-bo/postgres-deployment.yml

# PASSO 2: Se o rolling update entrar em deadlock (Multi-Attach error)
# Identificar o nome do ReplicaSet antigo e do novo
kubectl get replicasets -n default

# Forçar remoção do pod antigo que segura o PVC
kubectl delete pod <nome-do-pod-postgres-antigo> --force --grace-period=0 -n default

# Se o ReplicaSet antigo recriar o pod, deletar o novo pod também
kubectl delete pod <nome-do-pod-recriado> --force --grace-period=0 -n default

# Escalar o ReplicaSet antigo para 0 para impedir recriações
kubectl scale replicaset <nome-do-replicaset-antigo> --replicas=0 -n default

# PASSO 3: Aguardar o postgres ficar Running
kubectl rollout status deployment/postgres -n default

# PASSO 4: Forçar restart do kube-news (não esperar o back-off exponencial)
kubectl delete pod <nome-do-pod-kube-news> -n default

# PASSO 5: Verificar que ambos os pods estão Running
kubectl get pods -n default -o wide

# PASSO 6: Verificar conectividade da aplicação
curl -I http://134.209.130.140:80
```

### Armadilhas conhecidas

**1. O ReplicaSet antigo recria pods automaticamente**

Ao deletar o pod antigo do postgres com `--force`, o ReplicaSet antigo (que ainda tem `replicas: 1`) imediatamente recria um novo pod — que também vai tentar pegar o PVC e causar novo deadlock. É obrigatório escalar o ReplicaSet antigo para 0 logo após deletar seus pods:

```bash
kubectl scale replicaset <replicaset-antigo> --replicas=0 -n default
```

**2. Não fazer `kubectl rollout restart` sem corrigir o manifesto primeiro**

Reiniciar o Deployment sem corrigir o `subPath` resulta no mesmo erro. A ordem correta é: corrigir o manifesto → aplicar → tratar o deadlock.

**3. Verificar se o volume já tem dados antes de mudar o subPath**

Se o PostgreSQL tiver inicializado com sucesso em algum momento e houver dados no volume, adicionar `subPath: pgdata` muda o caminho de montagem e o postgres não encontrará os dados existentes (que estão na raiz do volume, não em `pgdata/`). Nesse caso, é necessário migrar os dados antes da mudança. **Neste incidente específico, o postgres nunca inicializou, então o volume estava vazio e não havia risco.**

**4. O kube-news não reconecta automaticamente enquanto está em back-off**

O back-off exponencial pode fazer o kube-news esperar vários minutos para tentar novamente, mesmo após o postgres estar disponível. Sempre forçar o restart do pod kube-news após confirmar que o postgres está `Running (1/1)`.

# Investigação do Incidente — 2026-05-16 — kubenews-nao-funciona

## Escopo
- Namespace: `default` (assumido, não especificado pelo usuário)
- Problema relatado: A aplicação Kubenews não está funcionando
- Investigação iniciada: 2026-05-16

---

## Estado do cluster no momento da investigação

### Pods (`kubectl get pods -n default -o wide`)

```
NAME                         READY   STATUS             RESTARTS      AGE   IP            NODE
kube-news-7649dfcf6b-ct79r   0/1     CrashLoopBackOff   3 (25s ago)   80s   10.109.0.28   pool-g1669fym6-33r5ds
postgres-85477b5999-scg4b    0/1     CrashLoopBackOff   3 (33s ago)   79s   10.109.0.31   pool-g1669fym6-33r5ds
```

Nenhum pod está `Running/Ready`. Ambos em `CrashLoopBackOff`.

### Recursos de suporte

```
NAME                        READY   UP-TO-DATE   AVAILABLE   IMAGES
deployment.apps/kube-news   0/1     1            0           fabricioveronez/imersao-kube-news:v1
deployment.apps/postgres    0/1     1            0           postgres:15-alpine

NAME               TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)
service/kube-news  LoadBalancer   10.108.57.195   134.209.130.140   80:30000/TCP
service/postgres   ClusterIP      10.108.49.107   <none>            5432/TCP

NAME                        STATUS   CAPACITY   STORAGECLASS
persistentvolumeclaim/postgres-pvc   Bound    1Gi   do-block-storage

NAME                   TYPE     DATA
secret/postgres-secret Opaque   3
```

O Secret `postgres-secret` existe e contém as 3 chaves esperadas (`db-database`, `db-username`, `db-password`).

---

## Evidências coletadas

### Pod: `kube-news-7649dfcf6b-ct79r`

#### Eventos relevantes (`kubectl describe pod`)

```
Warning  Failed   89s (x2 over 90s)   kubelet  spec.containers{kube-news}: Error: secret "postgres-secret" not found
Normal   Pulled   36s (x6 over 90s)   kubelet  Container image "fabricioveronez/imersao-kube-news:v1" already present
Normal   Created  36s (x4 over 75s)   kubelet  Container created
Normal   Started  36s (x4 over 75s)   kubelet  Container started
Warning  BackOff  30s (x10 over 71s)  kubelet  Back-off restarting failed container kube-news
```

> ATENÇÃO: os 2 primeiros eventos indicam que, no início do deploy, o secret `postgres-secret` NÃO EXISTIA ainda. O pod tentou iniciar antes do secret ser criado. Nas tentativas seguintes, o secret já existia e o container pôde iniciar — mas falhou por outro motivo (conexão recusada pelo postgres, que ainda não estava up).

#### Logs atuais e do crash anterior (idênticos — mesma falha)

```
Aplicação rodando na porta 8080
ConnectionRefusedError [SequelizeConnectionRefusedError]: connect ECONNREFUSED 10.108.49.107:5432
    at Client._connectionCallback (...)
  parent: Error: connect ECONNREFUSED 10.108.49.107:5432 {
    errno: -111,
    code: 'ECONNREFUSED',
    syscall: 'connect',
    address: 10.108.49.107,
    port: 5432
  }
Node.js v18.20.8
```

A aplicação sobe na porta 8080, mas falha imediatamente ao tentar conectar ao PostgreSQL em `10.108.49.107:5432` (ClusterIP do service `postgres`). A conexão é **recusada** — o postgres não está aceitando conexões porque ele próprio está em crash.

---

### Pod: `postgres-85477b5999-scg4b`

#### Eventos relevantes (`kubectl describe pod`)

```
Warning  FailedScheduling  90s  default-scheduler  0/2 nodes available: persistentvolumeclaim "postgres-pvc" not found
Warning  FailedScheduling  90s  default-scheduler  0/2 nodes available: pod has unbound immediate PersistentVolumeClaims
Warning  FailedScheduling  89s  default-scheduler  0/2 nodes available: pod has unbound immediate PersistentVolumeClaims
Normal   Scheduled         89s  default-scheduler  Successfully assigned to pool-g1669fym6-33r5ds
Normal   SuccessfulAttachVolume  86s  attachdetach-controller  AttachVolume.Attach succeeded for pvc-cc422e10...
Normal   Pulled   0s (x5 over 85s)  kubelet  Container image "postgres:15-alpine" already present
Normal   Created  0s (x5 over 85s)  kubelet  Container created
Normal   Started  0s (x5 over 85s)  kubelet  Container started
Warning  BackOff  <invalid> (x8)    kubelet  Back-off restarting failed container postgres
```

> ATENÇÃO: o scheduler recusou inicialmente o pod porque o PVC `postgres-pvc` não havia sido criado antes do Deployment. Após o PVC ser criado (bound), o pod foi agendado e o volume foi montado com sucesso — mas o container falha no initdb.

#### Logs atuais e do crash anterior (idênticos — mesma falha)

```
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.utf8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

initdb: error: directory "/var/lib/postgresql/data" exists but is not empty
initdb: detail: It contains a lost+found directory, perhaps due to it being a mount point.
initdb: hint: Using a mount point directly as the data directory is not recommended.
                Create a subdirectory under the mount point.
```

O `initdb` recusa inicializar porque o diretório `/var/lib/postgresql/data` já contém um subdiretório `lost+found` — artefato do sistema de arquivos do volume block storage (DigitalOcean `do-block-storage`) que formata o volume com ext4, deixando `lost+found` na raiz.

---

## Divergências repositório vs cluster

### 1. Manifesto `postgres-deployment.yml` — mountPath conflitante com block storage

**O que o manifesto diz:**
```yaml
volumeMounts:
  - name: postgres-data
    mountPath: /var/lib/postgresql/data
```

**O que está ocorrendo no cluster:**
O volume block storage é montado diretamente em `/var/lib/postgresql/data`. O sistema de arquivos ext4 cria `lost+found` na raiz do volume, impedindo o `initdb` de inicializar o cluster PostgreSQL.

**Correção necessária:** Adicionar `subPath: pgdata` no `volumeMount` para que o postgres use um subdiretório dentro do volume, evitando o conflito com `lost+found`.

```yaml
# CORRIGIDO
volumeMounts:
  - name: postgres-data
    mountPath: /var/lib/postgresql/data
    subPath: pgdata
```

### 2. Secret `postgres-secret` — existe no cluster mas com timing problem

O Secret está corretamente aplicado (annotation `last-applied-configuration` confirma os mesmos valores do manifesto `k8s-bo/postgres-secret.yml`). O erro inicial `secret "postgres-secret" not found` (2 eventos) indica que o Deployment foi aplicado antes do Secret estar disponível. Isso é um problema de **ordem de aplicação** dos manifestos, não de conteúdo.

**Manifesto k8s-bo/postgres-secret.yml:**
```yaml
stringData:
  db-database: kubedevnews
  db-username: kubedevnews
  db-password: "Pg#123"
```
**Cluster:** Mesmo conteúdo — Secret correto.

### 3. Diretório `k8s/` foi deletado do repositório

O git status mostra ` D k8s/kube-news.yml` — o arquivo original foi deletado. Os manifestos ativos estão em `k8s-bo/`, que é o que está rodando no cluster. Não há divergência de conteúdo entre `k8s-bo/` e o cluster, exceto o problema do `subPath` acima.

---

## Hipótese de causa raiz

### Causa raiz primária — PostgreSQL não inicializa (bloqueia toda a stack)

O volume `do-block-storage` (block storage da DigitalOcean, formatado em ext4) é montado diretamente em `/var/lib/postgresql/data`. O sistema de arquivos ext4 cria automaticamente um diretório `lost+found` na raiz do volume ao ser formatado. O `initdb` do PostgreSQL recusa inicializar qualquer diretório que **não esteja vazio**, resultando em exit code 1 a cada tentativa — causando o `CrashLoopBackOff` do pod postgres.

### Causa secundária em cascata — kube-news não consegue conectar ao postgres

Como o postgres nunca sobe (está em crash loop), o service `postgres` (ClusterIP `10.108.49.107:5432`) não tem endpoints ativos. A aplicação Node.js sobe na porta 8080 mas, ao tentar conectar ao banco via Sequelize, recebe `ECONNREFUSED` imediatamente. A aplicação termina com exit code 1, gerando seu próprio `CrashLoopBackOff`.

### Causa terciária — ordem de aplicação dos manifestos

Os eventos iniciais do pod `kube-news` registram `Error: secret "postgres-secret" not found` (2 ocorrências). Isso indica que os Deployments foram aplicados antes do Secret existir no cluster. Atualmente o Secret existe e esse erro não ocorre mais — mas a ordem de aplicação deve ser documentada para evitar reincidência.

**Cadeia de falhas:**
```
block storage ext4 cria lost+found
  → initdb recusa inicializar /var/lib/postgresql/data
    → postgres pod: CrashLoopBackOff (exit 1)
      → service postgres sem endpoints ativos
        → kube-news: ECONNREFUSED ao tentar conectar ao DB
          → kube-news pod: CrashLoopBackOff (exit 1)
            → aplicação indisponível para usuários
```

---

## Plano de correção

### Ação 1 — Corrigir o manifesto `postgres-deployment.yml` adicionando `subPath`

**Arquivo:** `/home/fabricioveronez/imersao/kube-news/k8s-bo/postgres-deployment.yml`

Adicionar `subPath: pgdata` ao `volumeMount` do container postgres:

```yaml
volumeMounts:
  - name: postgres-data
    mountPath: /var/lib/postgresql/data
    subPath: pgdata          # ← ADICIONAR ESTA LINHA
```

Isso faz o postgres usar `/var/lib/postgresql/data/pgdata` (subdiretório dentro do volume), onde não há `lost+found`, permitindo que o `initdb` inicialize com sucesso.

### Ação 2 — Re-aplicar o Deployment do postgres

```bash
kubectl apply -f /home/fabricioveronez/imersao/kube-news/k8s-bo/postgres-deployment.yml
```

Aguardar o pod entrar em `Running/Ready`:
```bash
kubectl rollout status deployment/postgres -n default
```

### Ação 3 — Verificar se kube-news se recupera automaticamente

Com o postgres up, o kube-news deve reconectar automaticamente no próximo restart do CrashLoop. Verificar:
```bash
kubectl rollout status deployment/kube-news -n default
kubectl logs -f deployment/kube-news -n default
```

### Ação 4 (preventiva) — Documentar ordem correta de aplicação dos manifestos

Para evitar o timing problem do Secret, a ordem recomendada de aplicação é:
1. `postgres-secret.yml`
2. `postgres-pvc.yml`
3. `postgres-service.yml`
4. `postgres-deployment.yml`
5. `app-service.yml`
6. `app-deployment.yml`

Considerar criar um script ou `kustomization.yaml` que enforça essa ordem.

### Ação 5 (opcional) — Remover arquivo `k8s/kube-news.yml` deletado do git

O `git status` mostra ` D k8s/kube-news.yml`. Fazer commit da deleção ou restaurar o arquivo se ainda for necessário.

---

## Sumário executivo

| Item | Detalhe |
|------|---------|
| Causa raiz | `initdb` falha porque volume ext4 contém `lost+found` em `/var/lib/postgresql/data` |
| Fix necessário | Adicionar `subPath: pgdata` no volumeMount do postgres deployment |
| Impacto | 100% — aplicação completamente indisponível |
| Risco da correção | Baixo — o volume ainda está vazio (postgres nunca inicializou) |
| Tempo estimado de correção | ~5 minutos |

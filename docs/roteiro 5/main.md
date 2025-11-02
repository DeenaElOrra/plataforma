# Minikube - Kubernetes Local

## Objetivo

Este guia descreve como executar todos os microserviços em um cluster Kubernetes local usando o **Minikube**, simulando um ambiente de orquestração completo em sua máquina de desenvolvimento. O projeto é composto por 6 microserviços (Gateway, Product, Order, Account, Auth e Exchange), além de serviços de infraestrutura (PostgreSQL e Redis), todos containerizados e orquestrados via Kubernetes.

---

## Pré-Requisitos

Antes de iniciar, certifique-se de ter as seguintes ferramentas instaladas:

- ✅ **Docker** instalado e em execução (versão >= 20.10)
- ✅ **Minikube** instalado (versão >= 1.0)
- ✅ **kubectl** instalado e configurado
- ✅ **Privilégios de sudo** (caso necessário para funcionalidades específicas)
- ✅ Pelo menos **4GB de RAM** disponível para o Minikube
- ✅ Pelo menos **20GB de espaço em disco**

### Instalação das Ferramentas

#### Docker
```bash
# macOS (Homebrew)
brew install --cask docker

# Linux
curl -fsSL https://get.docker.com | sh
```

#### Minikube
```bash
# macOS (Homebrew)
brew install minikube

# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

#### kubectl
```bash
# macOS (Homebrew)
brew install kubectl

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

---

## Passos para Iniciar

### 1. Iniciar o Minikube

No terminal, execute:

```bash
minikube start --driver=docker --memory=4096 --cpus=2
```

**Parâmetros:**
- `--driver=docker`: Utiliza o Docker como driver (também pode usar `virtualbox`, `hyperkit`, etc.)
- `--memory=4096`: Aloca 4GB de RAM para o cluster
- `--cpus=2`: Aloca 2 CPUs para o cluster

Aguarde até que o Minikube faça o download da imagem base, inicie o cluster e configure o ambiente.

**Saída esperada:**
```bash
😄  minikube v1.x.x on Darwin 13.x
✨  Using the docker driver based on user configuration
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🔥  Creating docker container (CPUs=2, Memory=4096MB) ...
🐳  Preparing Kubernetes v1.28.x on Docker 24.0.x ...
🔎  Verifying Kubernetes components...
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster
```

---

### 2. Clonar e Navegar até o Projeto

```bash
# Clonar o repositório (se ainda não tiver feito)
git clone https://github.com/seu-usuario/pma.2025.2.git
cd pma.2025.2
```

---

### 3. Criar Recursos de Infraestrutura

Antes de deployar os microserviços, precisamos criar os recursos compartilhados: **ConfigMaps**, **Secrets**, **PostgreSQL** e **Redis**.

#### 3.1. Criar ConfigMap do PostgreSQL

Crie o arquivo `k8s/postgres-configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-configmap
data:
  POSTGRES_DB: store
```

Aplique:
```bash
kubectl apply -f k8s/postgres-configmap.yaml
```

#### 3.2. Criar Secrets do PostgreSQL

Crie o arquivo `k8s/postgres-secrets.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secrets
type: Opaque
data:
  POSTGRES_USER: c3RvcmU=        # base64: store
  POSTGRES_PASSWORD: c3RvcmU=    # base64: store
```

**Nota:** Para gerar seu próprio base64:
```bash
echo -n "store" | base64
```

Aplique:
```bash
kubectl apply -f k8s/postgres-secrets.yaml
```

#### 3.3. Deployar PostgreSQL

Crie o arquivo `k8s/postgres-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:17.6
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: postgres-configmap
                  key: POSTGRES_DB
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: POSTGRES_PASSWORD
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
      volumes:
        - name: postgres-storage
          emptyDir: {}

---

apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  type: ClusterIP
  ports:
    - port: 5432
      protocol: TCP
      targetPort: 5432
  selector:
    app: postgres
```

Aplique:
```bash
kubectl apply -f k8s/postgres-deployment.yaml
```

#### 3.4. Deployar Redis

Crie o arquivo `k8s/redis-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
          resources:
            requests:
              memory: "128Mi"
              cpu: "50m"
            limits:
              memory: "256Mi"
              cpu: "200m"

---

apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    app: redis
spec:
  type: ClusterIP
  ports:
    - port: 6379
      protocol: TCP
      targetPort: 6379
  selector:
    app: redis
```

Aplique:
```bash
kubectl apply -f k8s/redis-deployment.yaml
```

---

### 4. Deployar os Microserviços

Agora vamos deployar todos os microserviços. Você pode fazer isso individualmente ou de uma vez.

#### Opção 1: Deploy Individual

```bash
# Gateway Service
kubectl apply -f api/gateway-service/k8s/k8s.yaml

# Product Service
kubectl apply -f api/product_service/k8s/k8s.yaml

# Order Service
kubectl apply -f api/order_service/k8s/k8s.yaml

# Account Service
kubectl apply -f api/accountservice/k8s/k8s.yaml

# Auth Service
kubectl apply -f api/auth_service/k8s/k8s.yaml

# Exchange Service
kubectl apply -f api/exchange/k8s/k8s.yaml
```

#### Opção 2: Deploy em Lote (Recomendado)

Crie um diretório `k8s/services/` e copie todos os manifests:

```bash
mkdir -p k8s/services
cp api/gateway-service/k8s/k8s.yaml k8s/services/gateway.yaml
cp api/product_service/k8s/k8s.yaml k8s/services/product.yaml
cp api/order_service/k8s/k8s.yaml k8s/services/order.yaml
cp api/accountservice/k8s/k8s.yaml k8s/services/account.yaml
cp api/auth_service/k8s/k8s.yaml k8s/services/auth.yaml
cp api/exchange/k8s/k8s.yaml k8s/services/exchange.yaml
```

Aplique todos de uma vez:
```bash
kubectl apply -f k8s/services/
```

---

### 5. Verificar Status

#### 5.1. Verificar Pods

```bash
kubectl get pods
```

**Saída esperada:**

```
NAME                                READY   STATUS    RESTARTS   AGE
postgres-XXXXXXXXXX-XXXXX           1/1     Running   0          5m
redis-XXXXXXXXXX-XXXXX              1/1     Running   0          5m
gateway-XXXXXXXXXX-XXXXX            1/1     Running   0          2m
product-service-XXXXXXXXXX-XXXXX    1/1     Running   0          2m
order-service-XXXXXXXXXX-XXXXX      1/1     Running   0          2m
account-service-XXXXXXXXXX-XXXXX    1/1     Running   0          2m
auth-service-XXXXXXXXXX-XXXXX       1/1     Running   0          2m
exchange-service-XXXXXXXXXX-XXXXX   1/1     Running   0          2m
```

![Minikube Pods Running](./docs/minikube-pods-running.png)

**Verificar se todos os pods estão com o STATUS `Running` e `READY 1/1`.**

**Caso algum pod esteja com status `CrashLoopBackOff` ou `Error`, inspecione os logs:**

```bash
kubectl logs <nome-do-pod>

# Exemplo:
kubectl logs product-service-XXXXXXXXXX-XXXXX
```

#### 5.2. Verificar Serviços

```bash
kubectl get svc
```

**Saída esperada:**

```
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP    10m
postgres           ClusterIP   10.96.0.10      <none>        5432/TCP   5m
redis              ClusterIP   10.96.0.11      <none>        6379/TCP   5m
gateway            ClusterIP   10.96.0.12      <none>        80/TCP     2m
product-service    ClusterIP   10.96.0.13      <none>        80/TCP     2m
order-service      ClusterIP   10.96.0.14      <none>        80/TCP     2m
account-service    ClusterIP   10.96.0.15      <none>        80/TCP     2m
auth-service       ClusterIP   10.96.0.16      <none>        80/TCP     2m
exchange-service   ClusterIP   10.96.0.17      <none>        80/TCP     2m
```

![Minikube Services Running](./docs/minikube-services-running.png)

---

## Acessando a Aplicação

### 6. Expor o Gateway para Acesso Externo

O Gateway é o ponto de entrada da aplicação. Para acessá-lo externamente, use o comando `minikube service`:

```bash
minikube service gateway
```

O Minikube criará automaticamente um túnel temporário e abrirá o serviço no navegador. Caso não abra automaticamente, será exibido o URL:

```
|-----------|---------|-------------|---------------------------|
| NAMESPACE |  NAME   | TARGET PORT |            URL            |
|-----------|---------|-------------|---------------------------|
| default   | gateway |          80 | http://192.168.49.2:30080 |
|-----------|---------|-------------|---------------------------|
🎉  Opening service default/gateway in default browser...
```

### 6.1. Alternativa: Port Forward

Se preferir usar port forwarding:

```bash
kubectl port-forward svc/gateway 8080:80
```

Acesse: `http://localhost:8080`

### 6.2. Alternativa: Minikube Tunnel (LoadBalancer)

Para usar serviços do tipo `LoadBalancer`, abra um novo terminal e execute:

```bash
minikube tunnel
```

Isso criará um túnel que expõe serviços LoadBalancer no `localhost`.

---

## Estrutura do Diretório k8s/

```
📁 k8s/
├── 📄 postgres-configmap.yaml       # ConfigMap do PostgreSQL
├── 📄 postgres-secrets.yaml         # Secrets do PostgreSQL
├── 📄 postgres-deployment.yaml      # Deployment e Service do PostgreSQL
├── 📄 redis-deployment.yaml         # Deployment e Service do Redis
├── 📁 services/
│   ├── 📄 gateway.yaml              # Gateway Service (porta 80)
│   ├── 📄 product.yaml              # Product Service (porta 80)
│   ├── 📄 order.yaml                # Order Service (porta 80)
│   ├── 📄 account.yaml              # Account Service (porta 80)
│   ├── 📄 auth.yaml                 # Auth Service (porta 80)
│   └── 📄 exchange.yaml             # Exchange Service (porta 80)
└── 📄 README.md                     # Este arquivo
```

---

## Microserviços Deployados

### 1. Gateway Service
- **Imagem:** `deenaelorra/gateway:latest`
- **Porta:** 8080 (container) → 80 (service)
- **Tipo:** ClusterIP
- **Dependências:** PostgreSQL, Redis
- **Recursos:** 200Mi-300Mi RAM, 50m-200m CPU

### 2. Product Service
- **Imagem:** `deenaelorra/product:latest`
- **Porta:** 8080 (container) → 80 (service)
- **Tipo:** ClusterIP
- **Dependências:** PostgreSQL, Redis
- **Recursos:** 200Mi-300Mi RAM, 50m-200m CPU
- **Schema:** `product` (Flyway auto-migration)

### 3. Order Service
- **Imagem:** `deenaelorra/order:latest`
- **Porta:** 8080 (container) → 80 (service)
- **Tipo:** ClusterIP
- **Dependências:** PostgreSQL, Redis
- **Recursos:** 200Mi-300Mi RAM, 50m-200m CPU

### 4. Account Service
- **Imagem:** `deenaelorra/account:latest`
- **Porta:** 8080 (container) → 80 (service)
- **Tipo:** ClusterIP
- **Dependências:** PostgreSQL, Redis
- **Recursos:** 200Mi-300Mi RAM, 50m-200m CPU

### 5. Auth Service
- **Imagem:** `deenaelorra/auth:latest`
- **Porta:** 8080 (container) → 80 (service)
- **Tipo:** ClusterIP
- **Dependências:** PostgreSQL, Redis
- **Recursos:** 200Mi-300Mi RAM, 50m-200m CPU
- **JWT:** Issuer: `store-auth`, Expiration: 1 hora

### 6. Exchange Service
- **Imagem:** `deenaelorra/exchange:latest`
- **Porta:** 8000 (container) → 80 (service)
- **Tipo:** ClusterIP
- **Dependências:** Nenhuma (Python/FastAPI)
- **Recursos:** 100Mi-200Mi RAM, 50m-200m CPU

---

## Infraestrutura

### PostgreSQL
- **Imagem:** `postgres:17.6`
- **Porta:** 5432
- **Database:** `store`
- **User:** `store` (configurado via Secret)
- **Password:** `store` (configurado via Secret)
- **Recursos:** 256Mi-512Mi RAM, 100m-500m CPU

### Redis
- **Imagem:** `redis:7-alpine`
- **Porta:** 6379
- **Recursos:** 128Mi-256Mi RAM, 50m-200m CPU

---

## Arquitetura de Comunicação

```
                          ┌──────────────────┐
                          │   Cliente/User   │
                          └────────┬─────────┘
                                   │
                                   v
                          ┌──────────────────┐
                          │     Gateway      │ :80
                          │   (API Gateway)  │
                          └────────┬─────────┘
                                   │
          ┌────────────┬───────────┼───────────┬────────────┐
          │            │           │           │            │
          v            v           v           v            v
    ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
    │ Product │  │  Order  │  │ Account │  │  Auth   │  │Exchange │
    │ Service │  │ Service │  │ Service │  │ Service │  │ Service │
    └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └─────────┘
         │            │            │            │
         └────────────┴────────────┴────────────┘
                      │            │
                      v            v
              ┌──────────────┐  ┌──────────┐
              │  PostgreSQL  │  │  Redis   │
              │   Database   │  │  Cache   │
              └──────────────┘  └──────────┘
```

---

## Comandos Úteis

### Gerenciar Cluster Minikube

```bash
# Ver status do Minikube
minikube status

# Parar o Minikube
minikube stop

# Deletar o cluster
minikube delete

# Acessar dashboard do Kubernetes
minikube dashboard

# Ver IP do Minikube
minikube ip

# SSH no node do Minikube
minikube ssh
```

### Gerenciar Recursos Kubernetes

```bash
# Listar todos os recursos
kubectl get all

# Descrever um pod
kubectl describe pod <nome-do-pod>

# Ver logs de um pod
kubectl logs <nome-do-pod>

# Ver logs em tempo real
kubectl logs -f <nome-do-pod>

# Executar comando em um pod
kubectl exec -it <nome-do-pod> -- /bin/bash

# Deletar um pod
kubectl delete pod <nome-do-pod>

# Deletar todos os recursos
kubectl delete -f k8s/services/

# Escalar um deployment
kubectl scale deployment gateway --replicas=3

# Ver eventos do cluster
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Gerenciar Imagens no Minikube

```bash
# Usar Docker do Minikube (evita pull do Docker Hub)
eval $(minikube docker-env)

# Voltar ao Docker local
eval $(minikube docker-env -u)

# Listar imagens no Minikube
minikube ssh docker images

# Carregar imagem local no Minikube
minikube image load deenaelorra/gateway:latest
```

---

## Troubleshooting

### Problema: Pods não iniciam (ImagePullBackOff)

**Causa:** Minikube não consegue baixar a imagem do Docker Hub

**Solução 1:** Use o Docker do Minikube para buildar localmente
```bash
eval $(minikube docker-env)
docker build -t deenaelorra/gateway:latest api/gateway-service/
```

**Solução 2:** Altere `imagePullPolicy` para `IfNotPresent` ou `Never`
```yaml
imagePullPolicy: IfNotPresent
```

### Problema: Pods reiniciando (CrashLoopBackOff)

**Causa:** Aplicação falhando ao iniciar

**Solução:** Verifique os logs
```bash
kubectl logs <nome-do-pod>
kubectl describe pod <nome-do-pod>
```

Possíveis causas:
- PostgreSQL não está pronto
- Credenciais inválidas
- Porta já em uso
- Falta de memória

### Problema: Não consigo acessar o serviço

**Solução:** Verifique se o serviço está rodando
```bash
kubectl get svc
minikube service gateway --url
```

Use port-forward como alternativa:
```bash
kubectl port-forward svc/gateway 8080:80
```

### Problema: PostgreSQL não está acessível

**Solução:** Verifique se o pod está rodando
```bash
kubectl get pods | grep postgres
kubectl logs <postgres-pod>
```

Teste a conexão:
```bash
kubectl exec -it <postgres-pod> -- psql -U store -d store
```

### Problema: Minikube lento

**Solução:** Aumente recursos
```bash
minikube stop
minikube delete
minikube start --driver=docker --memory=8192 --cpus=4
```

---

## Screenshots

### Dashboard do Kubernetes
![Kubernetes Dashboard](./docs/kubernetes-dashboard.png)

### Pods em Execução
![Pods Running](./docs/minikube-pods-running.png)

### Serviços Ativos
![Services Running](./docs/minikube-services-running.png)

### Logs de um Pod
![Pod Logs](./docs/minikube-pod-logs.png)

---

## Deploy dos Microsserviços em Kubernetes

### Resumo dos Manifests

| Serviço | Arquivo | Replicas | Imagem | Porta | Dependências |
|---------|---------|----------|--------|-------|--------------|
| Gateway | api/gateway-service/k8s/k8s.yaml | 1 | deenaelorra/gateway:latest | 8080→80 | PostgreSQL, Redis |
| Product | api/product_service/k8s/k8s.yaml | 1 | deenaelorra/product:latest | 8080→80 | PostgreSQL, Redis |
| Order | api/order_service/k8s/k8s.yaml | 1 | deenaelorra/order:latest | 8080→80 | PostgreSQL, Redis |
| Account | api/accountservice/k8s/k8s.yaml | 1 | deenaelorra/account:latest | 8080→80 | PostgreSQL, Redis |
| Auth | api/auth_service/k8s/k8s.yaml | 1 | deenaelorra/auth:latest | 8080→80 | PostgreSQL, Redis |
| Exchange | api/exchange/k8s/k8s.yaml | 1 | deenaelorra/exchange:latest | 8000→80 | Nenhuma |

---

## Repositórios

| Componente | Descrição | Link do Repositório |
|------------|-----------|---------------------|
| **Gateway Service** | API Gateway para roteamento | [Link do repositório] |
| **Product Service** | Gerenciamento de produtos | [Link do repositório] |
| **Order Service** | Gerenciamento de pedidos | [Link do repositório] |
| **Account Service** | Gerenciamento de contas | [Link do repositório] |
| **Auth Service** | Autenticação e JWT | [Link do repositório] |
| **Exchange Service** | Taxas de câmbio (Python) | [Link do repositório] |

---

## Tecnologias Utilizadas

- **Minikube** - Kubernetes local
- **kubectl** - CLI do Kubernetes
- **Docker** - Containerização
- **PostgreSQL 17.6** - Banco de dados relacional
- **Redis 7** - Cache em memória
- **Spring Boot 3.5.5** - Framework Java
- **FastAPI 0.104.1** - Framework Python

---

## Melhorias Futuras

1. **Helm Charts**: Simplificar deploy com Helm
2. **Horizontal Pod Autoscaler**: Escalonamento automático
3. **Ingress Controller**: Roteamento avançado
4. **Persistent Volumes**: Armazenamento persistente para PostgreSQL
5. **Monitoring**: Prometheus e Grafana
6. **Logging**: ELK Stack ou Loki
7. **Service Mesh**: Istio para observabilidade

---

## Licença

[Adicione informações sobre a licença do projeto]

---

## Contato

[Adicione informações de contato ou contribuição]

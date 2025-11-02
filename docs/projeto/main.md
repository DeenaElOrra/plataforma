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

![Minikube Pods Running](59829E57-DC1F-4EE3-844A-87CD93E67ACD.jpeg)

**Verificar se todos os pods estão com o STATUS `Running` e `READY 1/1`.**

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

![Minikube Services Running](cvcv.png)


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

## Repositórios

| Componente | Descrição | Link do Repositório |
|------------|-----------|---------------------|
| **Gateway Service** | API Gateway para roteamento | https://github.com/DeenaElOrra/gateway-service |
| **Product Service** | Gerenciamento de produtos | https://github.com/DeenaElOrra/product_service |
| **Order Service** | Gerenciamento de pedidos | https://github.com/DeenaElOrra/order_service |
| **Account Service** | Gerenciamento de contas | https://github.com/DeenaElOrra/accountservice |
| **Auth Service** | Autenticação e JWT | https://github.com/DeenaElOrra/auth_service |
| **Exchange Service** | Taxas de câmbio (Python) | https://github.com/DeenaElOrra/exchange |
| **Auth** | Auth | https://github.com/DeenaElOrra/auth |
| **PMA** | Tudo junto | https://github.com/DeenaElOrra/pma.2025.2 |
| **Account** | Account | https://github.com/DeenaElOrra/account |
| **Order** | Order | https://github.com/DeenaElOrra/order |



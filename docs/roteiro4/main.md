# Jenkins CI/CD - Documentação

## Visão Geral

O **Jenkins** é a plataforma de CI/CD (Continuous Integration/Continuous Deployment) utilizada neste projeto para automatizar o processo de build, teste e deploy dos microserviços. A configuração implementa pipelines declarativos para 10 componentes da aplicação (4 APIs de interface e 6 serviços), incluindo build de dependências, compilação Maven, criação de imagens Docker multi-plataforma (linux/amd64 e linux/arm64) e push automático para o Docker Hub. O Jenkins está containerizado com Docker Compose, inclui ferramentas como Maven, Docker CE e kubectl v1.30, e gerencia credenciais criptografadas para integração segura com o Docker Hub.

---

## Estrutura do Projeto Jenkins

```
📁 jenkins/
├── 📄 compose.yaml
└── 📁 config/
    └── 📁 jenkins/
        ├── 📄 config.xml
        ├── 📄 credentials.xml
        ├── 📁 jobs/
        │   ├── 📁 account/
        │   │   └── 📄 config.xml
        │   ├── 📁 accountservice/
        │   │   └── 📄 config.xml
        │   ├── 📁 auth/
        │   │   └── 📄 config.xml
        │   ├── 📁 auth-service/
        │   │   └── 📄 config.xml
        │   ├── 📁 exchange/
        │   │   └── 📄 config.xml
        │   ├── 📁 gateway/
        │   │   └── 📄 config.xml
        │   ├── 📁 order/
        │   │   └── 📄 config.xml
        │   ├── 📁 order-service/
        │   │   └── 📄 config.xml
        │   ├── 📁 product/
        │   │   └── 📄 config.xml
        │   └── 📁 product-service/
        │       └── 📄 config.xml
        └── 📁 plugins/
```

---

## Pipelines Configurados

### APIs de Interface (Libraries)

Estes jobs compilam as bibliotecas compartilhadas que serão usadas como dependências pelos serviços.

#### 1. Account API
**Jenkinsfile:** `api/account/Jenkinsfile`

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean install'
            }
        }
    }
}
```

**Funcionalidade:**
- Compila a biblioteca Account API
- Instala no repositório Maven local
- Pula testes para acelerar o build
- Será usada como dependência pelo Account Service

---

#### 2. Auth API
**Jenkinsfile:** `api/auth/Jenkinsfile`

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean install'
            }
        }
    }
}
```

**Funcionalidade:**
- Compila a biblioteca Auth API
- Instala no repositório Maven local
- Será usada como dependência pelo Auth Service

---

#### 3. Product API
**Jenkinsfile:** `api/product/Jenkinsfile`

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean install'
            }
        }
    }
}
```

**Funcionalidade:**
- Compila a biblioteca Product API (FeignClient)
- Instala no repositório Maven local
- Será usada como dependência pelo Product Service e Order Service

---

#### 4. Order API
**Jenkinsfile:** `api/order/Jenkinsfile`

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean install'
            }
        }
    }
}
```

**Funcionalidade:**
- Compila a biblioteca Order API (FeignClient)
- Instala no repositório Maven local
- Será usada como dependência pelo Order Service

---

### Serviços com Docker (Services)

Estes jobs compilam os serviços, criam imagens Docker multi-plataforma e fazem push para o Docker Hub.

#### 5. Product Service
**Jenkinsfile:** `api/product_service/Jenkinsfile`

```groovy
pipeline {
    agent any
    environment {
        SERVICE = 'product'
        NAME = "deenaelorra/${env.SERVICE}"
    }
    stages {
        stage('Dependencies') {
            steps {
                build job: 'product', wait: true
            }
        }
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }
        stage('Build & Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credential',
                    usernameVariable: 'USERNAME',
                    passwordVariable: 'TOKEN')])
                {
                    sh "docker login -u $USERNAME -p $TOKEN"
                    sh "docker buildx create --use --platform=linux/arm64,linux/amd64 --node multi-platform-builder-${env.SERVICE} --name multi-platform-builder-${env.SERVICE}"
                    sh "docker buildx build --platform=linux/arm64,linux/amd64 --push --tag ${env.NAME}:latest --tag ${env.NAME}:${env.BUILD_ID} -f Dockerfile ."
                    sh "docker buildx rm --force multi-platform-builder-${env.SERVICE}"
                }
            }
        }
    }
}
```

**Funcionalidade:**
- **Stage 1 - Dependencies**: Executa o build do job 'product' (API) e aguarda conclusão
- **Stage 2 - Build**: Compila o serviço com Maven (pula testes)
- **Stage 3 - Build & Push Image**:
  - Login no Docker Hub com credenciais seguras
  - Cria builder multi-plataforma (ARM64 + AMD64)
  - Gera imagem Docker para ambas as arquiteturas
  - Push para Docker Hub com tags `:latest` e `:BUILD_ID`
  - Remove o builder temporário

**Docker Hub:** `deenaelorra/product:latest` e `deenaelorra/product:{BUILD_ID}`

---

#### 6. Order Service
**Jenkinsfile:** `api/order_service/Jenkinsfile`

```groovy
pipeline {
    agent any
    environment {
        SERVICE = 'order'
        NAME = "deenaelorra/${env.SERVICE}"
    }
    stages {
        stage('Dependencies') {
            steps {
                build job: 'product', wait: true
                build job: 'order', wait: true
            }
        }
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }
        stage('Build & Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credential',
                    usernameVariable: 'USERNAME',
                    passwordVariable: 'TOKEN')])
                {
                    sh "docker login -u $USERNAME -p $TOKEN"
                    sh "docker buildx create --use --platform=linux/arm64,linux/amd64 --node multi-platform-builder-${env.SERVICE} --name multi-platform-builder-${env.SERVICE}"
                    sh "docker buildx build --platform=linux/arm64,linux/amd64 --push --tag ${env.NAME}:latest --tag ${env.NAME}:${env.BUILD_ID} -f Dockerfile ."
                    sh "docker buildx rm --force multi-platform-builder-${env.SERVICE}"
                }
            }
        }
    }
}
```

**Funcionalidade:**
- **Dependencies**: Compila 'product' e 'order' APIs sequencialmente
- **Build**: Compila o Order Service
- **Build & Push Image**: Gera e publica imagem Docker multi-plataforma

**Docker Hub:** `deenaelorra/order:latest` e `deenaelorra/order:{BUILD_ID}`

---

#### 7. Account Service
**Jenkinsfile:** `api/accountservice/Jenkinsfile`

```groovy
pipeline {
    agent any
    environment {
        SERVICE = 'accountservice'
        NAME = "deenaelorra/${env.SERVICE}"
    }
    stages {
        stage('Dependencies') {
            steps {
                build job: 'account', wait: true
            }
        }
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }
        stage('Build & Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credential',
                    usernameVariable: 'USERNAME',
                    passwordVariable: 'TOKEN')])
                {
                    sh "docker login -u $USERNAME -p $TOKEN"
                    sh "docker buildx create --use --platform=linux/arm64,linux/amd64 --node multi-platform-builder-${env.SERVICE} --name multi-platform-builder-${env.SERVICE}"
                    sh "docker buildx build --platform=linux/arm64,linux/amd64 --push --tag ${env.NAME}:latest --tag ${env.NAME}:${env.BUILD_ID} -f Dockerfile ."
                    sh "docker buildx rm --force multi-platform-builder-${env.SERVICE}"
                }
            }
        }
    }
}
```

**Docker Hub:** `deenaelorra/accountservice:latest` e `deenaelorra/accountservice:{BUILD_ID}`

---

#### 8. Auth Service
**Jenkinsfile:** `api/auth_service/Jenkinsfile`

```groovy
pipeline {
    agent any
    environment {
        SERVICE = 'auth-service'
        NAME = "deenaelorra/${env.SERVICE}"
    }
    stages {
        stage('Dependencies') {
            steps {
                build job: 'account', wait: true
                build job: 'auth', wait: true
            }
        }
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }
        stage('Build & Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credential',
                    usernameVariable: 'USERNAME',
                    passwordVariable: 'TOKEN')])
                {
                    sh "docker login -u $USERNAME -p $TOKEN"
                    sh "docker buildx create --use --platform=linux/arm64,linux/amd64 --node multi-platform-builder-${env.SERVICE} --name multi-platform-builder-${env.SERVICE}"
                    sh "docker buildx build --platform=linux/arm64,linux/amd64 --push --tag ${env.NAME}:latest --tag ${env.NAME}:${env.BUILD_ID} -f Dockerfile ."
                    sh "docker buildx rm --force multi-platform-builder-${env.SERVICE}"
                }
            }
        }
    }
}
```

**Docker Hub:** `deenaelorra/auth-service:latest` e `deenaelorra/auth-service:{BUILD_ID}`

---

#### 9. Gateway Service
**Jenkinsfile:** `api/gateway-service/Jenkinsfile`

```groovy
pipeline {
    agent any
    environment {
        SERVICE = 'gateway'
        NAME = "deenaelorra/${env.SERVICE}"
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }
        stage('Build & Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credential',
                    usernameVariable: 'USERNAME',
                    passwordVariable: 'TOKEN')])
                {
                    sh "docker login -u $USERNAME -p $TOKEN"
                    sh "docker buildx create --use --platform=linux/arm64,linux/amd64 --node multi-platform-builder-${env.SERVICE} --name multi-platform-builder-${env.SERVICE}"
                    sh "docker buildx build --platform=linux/arm64,linux/amd64 --push --tag ${env.NAME}:latest --tag ${env.NAME}:${env.BUILD_ID} -f Dockerfile ."
                    sh "docker buildx rm --force multi-platform-builder-${env.SERVICE}"
                }
            }
        }
    }
}
```

**Funcionalidade:**
- **Sem Dependencies**: Gateway não depende de outras APIs
- Compila e gera imagem Docker multi-plataforma

**Docker Hub:** `deenaelorra/gateway:latest` e `deenaelorra/gateway:{BUILD_ID}`

---

#### 10. Exchange Service (Python)
**Jenkinsfile:** `api/exchange/Jenkinsfile`

```groovy
pipeline {
    agent any
    environment {
        SERVICE = 'exchange'
        NAME = "deenaelorra/${env.SERVICE}"
    }
    stages {
        stage('Build & Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credential',
                    usernameVariable: 'USERNAME',
                    passwordVariable: 'TOKEN')])
                {
                    sh "docker login -u $USERNAME -p $TOKEN"
                    sh "docker buildx create --use --platform=linux/arm64,linux/amd64 --node multi-platform-builder-${env.SERVICE} --name multi-platform-builder-${env.SERVICE}"
                    sh "docker buildx build --platform=linux/arm64,linux/amd64 --push --tag ${env.NAME}:latest --tag ${env.NAME}:${env.BUILD_ID} -f Dockerfile ."
                    sh "docker buildx rm --force multi-platform-builder-${env.SERVICE}"
                }
            }
        }
    }
}
```

**Funcionalidade:**
- **Sem Maven Build**: Aplicação Python/FastAPI não requer compilação Java
- Apenas cria e publica imagem Docker multi-plataforma

**Docker Hub:** `deenaelorra/exchange:latest` e `deenaelorra/exchange:{BUILD_ID}`

---

## Grafo de Dependências

```
┌─────────────────────────────────────────────────────────┐
│                    JENKINS PIPELINE                     │
└─────────────────────────────────────────────────────────┘

┌──────────────┐
│  account API │ ────┐
└──────────────┘     │
                     ├──> ┌──────────────────┐
                     │    │ accountservice   │ ──> Docker Hub
                     │    └──────────────────┘
┌──────────────┐     │
│   auth API   │ ────┤
└──────────────┘     │
                     └──> ┌──────────────────┐
                          │  auth-service    │ ──> Docker Hub
                          └──────────────────┘

┌──────────────┐
│ product API  │ ────┬──> ┌──────────────────┐
└──────────────┘     │    │ product-service  │ ──> Docker Hub
                     │    └──────────────────┘
┌──────────────┐     │
│  order API   │ ────┴──> ┌──────────────────┐
└──────────────┘          │  order-service   │ ──> Docker Hub
                          └──────────────────┘

                          ┌──────────────────┐
                          │     gateway      │ ──> Docker Hub
                          └──────────────────┘

                          ┌──────────────────┐
                          │    exchange      │ ──> Docker Hub
                          └──────────────────┘
```

---

## Configuração do Jenkins com Docker Compose

### Arquivo: `jenkins/compose.yaml`

```yaml
# docker compose up -d --build --force-recreate
name: ops

services:

  jenkins:
    container_name: jenkins
    build:
      dockerfile_inline: |
        FROM jenkins/jenkins:jdk21
        USER root

        # Install tools
        RUN apt-get update && apt-get install -y lsb-release iputils-ping maven

        # Install Docker
        RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
          https://download.docker.com/linux/debian/gpg
        RUN echo "deb [arch=$(dpkg --print-architecture) \
          signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
          https://download.docker.com/linux/debian \
          $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
        RUN apt-get update && apt-get install -y docker-ce

        # Install kubectl
        RUN apt-get install -y apt-transport-https ca-certificates curl
        RUN curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
        RUN chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
        RUN echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
        RUN chmod 644 /etc/apt/sources.list.d/kubernetes.list
        RUN apt-get update && apt-get install -y kubectl

        RUN usermod -aG docker jenkins
    ports:
      - 9080:8080
    volumes:
      - ${CONFIG:-./config}/jenkins:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    restart: always
```

**Características:**
- **Imagem Base:** jenkins/jenkins:jdk21
- **Porta:** 9080 (host) → 8080 (container)
- **Ferramentas Instaladas:**
  - Maven (builds Java)
  - Docker CE (builds de imagens)
  - kubectl v1.30 (deploy Kubernetes)
- **Volumes:**
  - `/var/jenkins_home` - Dados persistentes do Jenkins
  - `/var/run/docker.sock` - Socket Docker (Docker-in-Docker)
- **Restart Policy:** always

### Executar Jenkins

```bash
cd jenkins
docker compose up -d --build --force-recreate
```

Acesse: `http://localhost:9080`

---

## Screenshots do Jenkins

### Dashboard Principal
![Jenkins Dashboard](./docs/jenkins-dashboard.png)

### Lista de Jobs/Pipelines
![Jenkins Jobs](./docs/jenkins-jobs.png)

### Pipeline em Execução
![Jenkins Pipeline Running](./docs/jenkins-pipeline-running.png)

### Build Concluído com Sucesso
![Jenkins Build Success](./docs/jenkins-build-success.png)

---

## Credenciais do Docker Hub

### Configuração no Jenkins

As credenciais do Docker Hub são armazenadas de forma segura no Jenkins:

**Credential ID:** `dockerhub-credential`

**Tipo:** Username with password

**Campos:**
- **Username:** deenaelorra
- **Password:** [Criptografado]

### Screenshot da Configuração
![Docker Hub Credentials](./docs/jenkins-dockerhub-credentials.png)

### Como Adicionar Credenciais

1. Acesse Jenkins → Manage Jenkins → Credentials
2. Clique em "(global)" domain
3. Add Credentials
4. Preencha:
   - **Kind:** Username with password
   - **Scope:** Global
   - **Username:** deenaelorra
   - **Password:** [sua senha do Docker Hub]
   - **ID:** dockerhub-credential
   - **Description:** Docker Hub Credentials
5. Salvar

---

## Imagens Docker Publicadas

Todas as imagens são publicadas no Docker Hub com duas tags:

| Serviço | Repositório Docker Hub | Tags |
|---------|------------------------|------|
| Product Service | `deenaelorra/product` | `:latest`, `:{BUILD_ID}` |
| Order Service | `deenaelorra/order` | `:latest`, `:{BUILD_ID}` |
| Account Service | `deenaelorra/accountservice` | `:latest`, `:{BUILD_ID}` |
| Auth Service | `deenaelorra/auth-service` | `:latest`, `:{BUILD_ID}` |
| Gateway | `deenaelorra/gateway` | `:latest`, `:{BUILD_ID}` |
| Exchange | `deenaelorra/exchange` | `:latest`, `:{BUILD_ID}` |

**Plataformas Suportadas:**
- linux/amd64 (x86-64)
- linux/arm64 (ARM64)

---

## Build Multi-Plataforma com Docker Buildx

### Comandos Utilizados

```bash
# 1. Login no Docker Hub
docker login -u $USERNAME -p $TOKEN

# 2. Criar builder multi-plataforma
docker buildx create --use \
  --platform=linux/arm64,linux/amd64 \
  --node multi-platform-builder-{SERVICE} \
  --name multi-platform-builder-{SERVICE}

# 3. Build e push para ambas as plataformas
docker buildx build \
  --platform=linux/arm64,linux/amd64 \
  --push \
  --tag deenaelorra/{SERVICE}:latest \
  --tag deenaelorra/{SERVICE}:{BUILD_ID} \
  -f Dockerfile .

# 4. Remover builder temporário
docker buildx rm --force multi-platform-builder-{SERVICE}
```

**Benefícios:**
- Imagens funcionam em servidores AMD64 (Intel/AMD)
- Imagens funcionam em Apple Silicon (M1/M2/M3)
- Compatibilidade com ARM servers (AWS Graviton, etc.)

---

## Fluxo de CI/CD

### 1. Commit no Git
```
Developer → git push → GitHub
```

### 2. Trigger do Jenkins
```
GitHub Webhook → Jenkins → Start Pipeline
```

### 3. Execução da Pipeline

**Para Libraries (account, auth, product, order):**
```
┌──────────────────┐
│   Stage: Build   │
│   mvn install    │
└──────────────────┘
         │
         v
    [Success]
```

**Para Services (product-service, order-service, etc.):**
```
┌──────────────────────┐
│ Stage: Dependencies  │
│  Build lib jobs      │
│    (wait: true)      │
└──────────────────────┘
         │
         v
┌──────────────────────┐
│   Stage: Build       │
│  mvn package         │
└──────────────────────┘
         │
         v
┌──────────────────────┐
│ Stage: Build & Push  │
│  - Docker login      │
│  - buildx create     │
│  - buildx build      │
│  - Push to Hub       │
│  - buildx rm         │
└──────────────────────┘
         │
         v
   [Docker Hub]
```

### 4. Deploy (Manual ou Automático)
```
Docker Hub → kubectl apply → Kubernetes Cluster
```

---

## Jobs Configurados

### Resumo de Todos os Jobs

| Job Name | Tipo | Dependências | Docker Image | Plataformas |
|----------|------|--------------|--------------|-------------|
| account | Library | - | - | - |
| accountservice | Service | account | deenaelorra/accountservice | amd64, arm64 |
| auth | Library | - | - | - |
| auth-service | Service | account, auth | deenaelorra/auth-service | amd64, arm64 |
| product | Library | - | - | - |
| product-service | Service | product | deenaelorra/product | amd64, arm64 |
| order | Library | - | - | - |
| order-service | Service | product, order | deenaelorra/order | amd64, arm64 |
| gateway | Service | - | deenaelorra/gateway | amd64, arm64 |
| exchange | Service | - | deenaelorra/exchange | amd64, arm64 |

---

## Configuração de Segurança

### Autenticação e Autorização

**Arquivo:** `jenkins/config/jenkins/config.xml`

```xml
<authorizationStrategy class="hudson.security.FullControlOnceLoggedInAuthorizationStrategy">
  <denyAnonymousReadAccess>true</denyAnonymousReadAccess>
</authorizationStrategy>
```

**Características:**
- Acesso anônimo negado
- Usuários autenticados têm controle total
- CSRF protection habilitado

### Credenciais Criptografadas

**Arquivo:** `jenkins/config/jenkins/credentials.xml`

As credenciais são armazenadas com criptografia AES:
- Username: plain text
- Password: criptografado com secret key do Jenkins
- Secret key: armazenado em `jenkins/config/jenkins/secrets/`

---

## Comandos Úteis

### Gerenciar Jenkins Container

```bash
# Iniciar Jenkins
cd jenkins
docker compose up -d

# Ver logs
docker compose logs -f jenkins

# Parar Jenkins
docker compose down

# Rebuild completo
docker compose up -d --build --force-recreate

# Entrar no container
docker exec -it jenkins bash
```

### Gerenciar Builds

```bash
# Trigger manual de build (via CLI)
java -jar jenkins-cli.jar -s http://localhost:9080/ build product-service

# Ver status do último build
java -jar jenkins-cli.jar -s http://localhost:9080/ get-job product-service
```

### Docker Hub

```bash
# Pull de uma imagem
docker pull deenaelorra/product:latest

# Ver tags disponíveis
curl -L -s 'https://registry.hub.docker.com/v2/repositories/deenaelorra/product/tags?page_size=100' | jq -r '.results[].name'

# Run local
docker run -p 8080:8080 deenaelorra/product:latest
```

---

## Troubleshooting

### Problema: Build falha por falta de dependência

**Solução:** Certifique-se de que os jobs das libraries foram executados primeiro:
```bash
# Ordem correta:
1. account → accountservice
2. auth → auth-service
3. product → product-service
4. product + order → order-service
```

### Problema: Docker Hub push falha

**Solução:** Verifique as credenciais:
1. Jenkins → Manage Jenkins → Credentials
2. Verifique se `dockerhub-credential` existe
3. Teste login manual: `docker login -u deenaelorra`

### Problema: Multi-platform build falha

**Solução:** Verifique se buildx está instalado:
```bash
docker exec -it jenkins bash
docker buildx version
```

### Problema: Permissão negada ao acessar Docker

**Solução:** Verifique se Jenkins está no grupo docker:
```bash
docker exec -it jenkins bash
groups jenkins  # deve incluir 'docker'
```

---

## Melhorias Futuras

1. **Testes Automatizados**: Adicionar stage de testes (`mvn test`)
2. **SonarQube**: Integração para análise de código
3. **Notificações**: Slack/Email para builds falhados
4. **Deploy Automático**: kubectl apply após push bem-sucedido
5. **Rollback**: Pipeline de rollback para versões anteriores
6. **Cache de Dependências**: Melhorar performance com cache Maven

---

## Tecnologias Utilizadas

- **Jenkins 2.533** - Servidor CI/CD
- **JDK 21** - Ambiente Java
- **Maven** - Build tool para Java
- **Docker CE** - Containerização
- **Docker Buildx** - Builds multi-plataforma
- **kubectl v1.30** - Deploy Kubernetes
- **Git** - Controle de versão
- **Docker Hub** - Registry de imagens

---

## Repositórios

| Componente | Descrição | Link do Repositório |
|------------|-----------|---------------------|
| **Jenkins Config** | Configurações e pipelines do Jenkins | [Link do repositório] |
| **Docker Hub** | Registry com todas as imagens | https://hub.docker.com/u/deenaelorra |

---

## Licença

[Adicione informações sobre a licença do projeto]

---

## Contato

[Adicione informações de contato ou contribuição]

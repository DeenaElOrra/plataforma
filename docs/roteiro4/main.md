# Jenkins CI/CD - Documentação

## Visão Geral

O **Jenkins** é a plataforma de CI/CD (Continuous Integration/Continuous Deployment) utilizada neste projeto para automatizar o processo de build, teste e deploy dos microserviços. A configuração implementa pipelines declarativos para 10 componentes da aplicação (4 APIs de interface e 6 serviços), incluindo build de dependências, compilação Maven, criação de imagens Docker multi-plataforma (linux/amd64 e linux/arm64) e push automático para o Docker Hub. 

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

### Executar Jenkins

```bash
cd jenkins
docker compose up -d --build --force-recreate
```

Acesse: `http://localhost:9080`

---

## Screenshots do Jenkins

### Dashboard Principal
![Jenkins Dashboard](adeded.png)

---

## Credenciais do Docker Hub
![Docker Hub Credentials](asd.png)


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


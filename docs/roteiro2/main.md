# Exchange API - Documentação

## Visão Geral

A **Exchange API** é uma API RESTful desenvolvida em Python com FastAPI para gerenciamento de taxas de câmbio em tempo real. A API integra-se com o serviço externo [Awesome API](https://economia.awesomeapi.com.br) para obter cotações atualizadas de moedas e aplica um spread de 1.5% para cálculo das taxas de compra e venda. O serviço é protegido por autenticação JWT através do API Gateway e requer um ID de conta válido em cada requisição. A API suporta operações assíncronas para melhor performance e é totalmente containerizada com Docker e Kubernetes.

---

## Endpoints Principais

### 1. Obter Taxa de Câmbio
**Endpoint:** `GET /exchange/{from_currency}/{to_currency}`

**Descrição:** Retorna as taxas de compra e venda entre duas moedas.

**Headers Obrigatórios:**
```
Authorization: Bearer {jwt_token}
id-account: {account_id}
```

**Parâmetros de Path:**
- `from_currency`: Código da moeda de origem (3 letras, ex: USD, BRL, EUR)
- `to_currency`: Código da moeda de destino (3 letras, ex: USD, BRL, EUR)

**Exemplo:** `GET /exchange/USD/BRL`

**Response (200 OK):**
```json
{
  "buy": 5.2575,
  "sell": 5.1825,
  "timestamp": "2025-10-31 14:30:45",
  "account_id": "550e8400-e29b-41d4-a716-446655440000",
  "base_rate": 5.22
}
```

**Campos da Resposta:**
- `buy`: Taxa de compra com spread de 1.5% aplicado
- `sell`: Taxa de venda com spread de 1.5% aplicado
- `timestamp`: Data e hora da transação
- `account_id`: ID da conta do usuário autenticado
- `base_rate`: Taxa base sem spread (obtida da API externa)

**Códigos de Status:**
- `200 OK`: Taxa de câmbio retornada com sucesso
- `400 Bad Request`: Códigos de moeda inválidos ou par não suportado
- `401 Unauthorized`: Account ID ausente ou inválido
- `503 Service Unavailable`: API externa indisponível
- `504 Gateway Timeout`: Timeout ao chamar API externa
- `500 Internal Server Error`: Erro interno do servidor

---

### 2. Health Check
**Endpoint:** `GET /health`

**Descrição:** Verifica o status de saúde do serviço.

**Autenticação:** Não requerida

**Response (200 OK):**
```json
{
  "status": "healthy",
  "service": "exchange-api",
  "timestamp": "2025-10-31T14:30:45.123456"
}
```

---

### 3. Informações da API
**Endpoint:** `GET /`

**Descrição:** Retorna informações sobre a API e seus endpoints disponíveis.

**Autenticação:** Não requerida

**Response (200 OK):**
```json
{
  "service": "Currency Exchange API",
  "version": "1.0.0",
  "endpoints": {
    "exchange": "/exchange/{from}/{to}",
    "health": "/health",
    "docs": "/docs"
  }
}
```

---

## Testes com Postman

### Obter Taxa de Câmbio (USD para BRL)
![GET /exchange/USD/BRL](./docs/postman-exchange-usd-brl.png)

### Obter Taxa de Câmbio (EUR para BRL)
![GET /exchange/EUR/BRL](./docs/postman-exchange-eur-brl.png)

### Health Check
![GET /health](./docs/postman-health.png)

### Erro - Moeda Inválida
![GET /exchange/INVALID/BRL](./docs/postman-error-invalid-currency.png)

---

## Estrutura do Projeto

### Exchange API (FastAPI)
```
📁 api/
└── 📁 exchange/
    ├── 📁 app/
    │   ├── 📄 main.py
    │   └── 📄 requirements.txt
    ├── 📁 k8s/
    │   └── 📄 k8s.yaml
    ├── 📄 Dockerfile
    ├── 📄 README.md
    └── 📁 .git/
```

### Detalhamento dos Arquivos

**main.py** - Aplicação FastAPI principal contendo:
- `ExchangeRateResponse`: Modelo de resposta (Pydantic)
- `ExchangeRateService`: Serviço de lógica de negócios
- Rotas e endpoints REST
- Validações de entrada
- Tratamento de erros

**requirements.txt** - Dependências Python:
- `fastapi==0.104.1` - Framework web
- `uvicorn[standard]==0.24.0` - Servidor ASGI
- `httpx==0.25.1` - Cliente HTTP assíncrono
- `pydantic==2.5.0` - Validação de dados

**Dockerfile** - Containerização da aplicação:
- Base: `python:3.11-slim`
- Porta exposta: 8000
- Comando: `uvicorn app.main:app --host 0.0.0.0 --port 8000`

**k8s.yaml** - Configuração Kubernetes:
- Deployment com 1 réplica
- Service ClusterIP na porta 80
- Recursos: 100Mi-200Mi RAM, 50m-200m CPU

---

## Repositórios

| Componente | Descrição | Link do Repositório |
|------------|-----------|---------------------|
| **Exchange API** | API de taxas de câmbio em tempo real (FastAPI) | [Link do repositório] |
| **Gateway Service** | API Gateway para roteamento e autenticação | [Link do repositório] |

---

## Tecnologias Utilizadas

- **FastAPI 0.104.1** - Framework web moderno e de alta performance
- **Python 3.11** - Linguagem de programação
- **Uvicorn** - Servidor ASGI de alto desempenho
- **Httpx 0.25.1** - Cliente HTTP assíncrono para chamadas externas
- **Pydantic 2.5.0** - Validação de dados e serialização
- **Docker** - Containerização da aplicação
- **Kubernetes** - Orquestração de containers

---

## Configuração e Execução

### Pré-requisitos
- Python 3.11+
- Docker (opcional, para containerização)
- Kubernetes (opcional, para deploy em cluster)

### Executar Localmente (sem Docker)

```bash
cd api/exchange
pip install -r app/requirements.txt
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

A aplicação estará disponível em: `http://localhost:8000`

### Executar com Docker

```bash
cd api/exchange
docker build -t exchange-api .
docker run -p 8000:8000 exchange-api
```

### Deploy no Kubernetes

```bash
cd api/exchange
kubectl apply -f k8s/k8s.yaml
```

O serviço estará disponível internamente no cluster em: `http://exchange:80`

---

## Integração com API Externa

### Awesome API - Economia
- **URL Base:** `https://economia.awesomeapi.com.br/json/last`
- **Formato de Requisição:** `GET /{FROM_CURRENCY}-{TO_CURRENCY}`
- **Exemplo:** `https://economia.awesomeapi.com.br/json/last/USD-BRL`
- **Timeout:** 10 segundos
- **Taxa Utilizada:** Campo `bid` da resposta (taxa de compra)

### Cálculo de Spread

A API aplica um spread de **1.5%** sobre a taxa base:

```python
SPREAD_PERCENTAGE = 0.015

buy_rate = base_rate * (1 + SPREAD_PERCENTAGE)   # +1.5%
sell_rate = base_rate * (1 - SPREAD_PERCENTAGE)  # -1.5%
```

**Exemplo:**
- Taxa base: 5.22
- Taxa de compra: 5.22 × 1.015 = **5.2575**
- Taxa de venda: 5.22 × 0.985 = **5.1825**

---

## Autenticação e Segurança

### Fluxo de Autenticação

1. **Cliente envia requisição** para o Gateway:
   ```
   GET /exchange/USD/BRL
   Authorization: Bearer {jwt_token}
   ```

2. **Gateway (AuthorizationFilter)** valida o token:
   - Extrai o token JWT do header `Authorization`
   - Valida o formato (`Bearer {token}`)
   - Chama o Auth Service para validar o token

3. **Auth Service** retorna o Account ID:
   ```json
   { "idAccount": "550e8400-e29b-41d4-a716-446655440000" }
   ```

4. **Gateway adiciona header** e encaminha:
   ```
   GET /exchange/USD/BRL
   id-account: 550e8400-e29b-41d4-a716-446655440000
   ```

5. **Exchange API** valida e processa:
   - Valida presença do header `id-account`
   - Valida códigos de moeda
   - Busca taxa de câmbio
   - Retorna resposta com dados do usuário

### Endpoints Públicos (sem autenticação)
- `GET /health` - Health check
- `GET /` - Informações da API
- `GET /docs` - Documentação interativa (Swagger)

---

## Validações

### Validação de Account ID
```python
def validate_account_id(account_id: Optional[str]) -> str:
    if not account_id or account_id.strip() == "":
        raise HTTPException(
            status_code=401,
            detail="Account ID is required"
        )
    return account_id
```

### Validação de Código de Moeda
```python
def validate_currency_code(currency: str) -> str:
    if not currency or len(currency) != 3 or not currency.isalpha():
        raise HTTPException(
            status_code=400,
            detail=f"Invalid currency code: {currency}"
        )
    return currency.upper()
```

---

## Tratamento de Erros

| Erro | Status Code | Descrição |
|------|-------------|-----------|
| Account ID ausente | 401 | Header `id-account` não encontrado |
| Código de moeda inválido | 400 | Código não tem 3 letras ou não é alfabético |
| Moedas idênticas | 400 | `from_currency` igual a `to_currency` |
| Par não suportado | 400 | API externa não suporta o par de moedas |
| Timeout API externa | 504 | Tempo limite de 10s excedido |
| API externa indisponível | 503 | Erro HTTP da API externa |
| Erro interno | 500 | Exceção não tratada |

---

## Arquitetura de Microserviços

### Comunicação com Gateway

```
Cliente
  |
  v
[Gateway Service - Spring Cloud Gateway] :8080
  |
  ├── Valida JWT token
  ├── Extrai Account ID do Auth Service
  ├── Adiciona header: id-account
  |
  v
[Exchange Service - FastAPI] :80
  |
  ├── Valida Account ID
  ├── Valida códigos de moeda
  ├── Chama API Externa (Awesome API)
  ├── Aplica spread de 1.5%
  └── Retorna taxas de compra/venda
```

### Configuração de Rota no Gateway

**Arquivo:** `api/gateway-service/src/main/resources/application.yaml`

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: exchange
          uri: http://exchange:80
          predicates:
            - Path=/exchange/**
```

---

## Documentação Interativa

A API fornece documentação interativa gerada automaticamente pelo FastAPI:

- **Swagger UI:** `http://localhost:8000/docs`
- **ReDoc:** `http://localhost:8000/redoc`

Essas interfaces permitem testar os endpoints diretamente no navegador.

---

## Logging

A aplicação utiliza o módulo `logging` do Python para registrar eventos:

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)
```

**Eventos logados:**
- Requisições recebidas
- Chamadas à API externa
- Erros e exceções
- Timeouts e falhas de comunicação

---

## Recursos do Kubernetes

### Deployment
- **Replicas:** 1
- **Image:** `deenaelorra/exchange:latest`
- **Container Port:** 8000
- **Resources:**
  - **Requests:** 100Mi RAM, 50m CPU
  - **Limits:** 200Mi RAM, 200m CPU

### Service
- **Type:** ClusterIP (acesso interno ao cluster)
- **Port:** 80 (externo) → 8000 (container)
- **Selector:** `app: exchange`

---

## Exemplos de Uso

### Obter Taxa USD para BRL

**Request:**
```bash
curl -X GET "http://localhost:8000/exchange/USD/BRL" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "id-account: 550e8400-e29b-41d4-a716-446655440000"
```

**Response:**
```json
{
  "buy": 5.2575,
  "sell": 5.1825,
  "timestamp": "2025-10-31 14:30:45",
  "account_id": "550e8400-e29b-41d4-a716-446655440000",
  "base_rate": 5.22
}
```

### Obter Taxa EUR para USD

**Request:**
```bash
curl -X GET "http://localhost:8000/exchange/EUR/USD" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "id-account: 550e8400-e29b-41d4-a716-446655440000"
```

**Response:**
```json
{
  "buy": 1.0959,
  "sell": 1.0791,
  "timestamp": "2025-10-31 14:32:10",
  "account_id": "550e8400-e29b-41d4-a716-446655440000",
  "base_rate": 1.0875
}
```

---

## Monitoramento

### Health Check Endpoint

O endpoint `/health` pode ser usado para:
- **Liveness Probe** no Kubernetes
- **Readiness Probe** no Kubernetes
- **Monitoramento externo** (uptime checks)

**Exemplo de Liveness Probe:**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 30
```

---

## Moedas Suportadas

A API suporta qualquer par de moedas disponível na Awesome API, incluindo:

### Principais Moedas
- **USD** - Dólar Americano
- **BRL** - Real Brasileiro
- **EUR** - Euro
- **GBP** - Libra Esterlina
- **JPY** - Iene Japonês
- **AUD** - Dólar Australiano
- **CAD** - Dólar Canadense
- **CHF** - Franco Suíço
- **ARS** - Peso Argentino
- **BTC** - Bitcoin

Para ver a lista completa de moedas suportadas, consulte: https://docs.awesomeapi.com.br

---

## Limitações e Considerações

1. **Rate Limiting:** A API externa pode ter limites de requisições
2. **Disponibilidade:** Dependente da disponibilidade da Awesome API
3. **Timeout:** Requisições com mais de 10 segundos são canceladas
4. **Autenticação Obrigatória:** Todos os endpoints de câmbio requerem autenticação
5. **Spread Fixo:** O spread de 1.5% é fixo e não configurável via API

---

## Licença

[Adicione informações sobre a licença do projeto]

---

## Contato

[Adicione informações de contato ou contribuição]

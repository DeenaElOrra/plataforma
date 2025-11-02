# Order API - Documentação

## Visão Geral

A **Order API** é uma API RESTful de CRUD (Create, Read, Update, Delete) desenvolvida em Spring Boot para gerenciamento de pedidos em um sistema de e-commerce. O projeto segue uma arquitetura de microserviços, onde o **Order** atua como um FeignClient que comunica com o **Order Service**, responsável pela lógica de negócios e persistência de dados em PostgreSQL. A API permite criar, listar, buscar e deletar pedidos através de endpoints HTTP, integrando-se com o Product Service para enriquecer os itens do pedido com informações de produto em tempo real. O serviço suporta multi-tenancy baseado em headers (id-account), utiliza migrações de banco de dados com Flyway, e implementa relacionamentos cascata entre pedidos e itens.

---

## Endpoints Principais

### 1. Criar Pedido
**Endpoint:** `POST /order`

**Descrição:** Cria um novo pedido com seus itens. Automaticamente busca informações dos produtos e calcula totais.

**Headers Obrigatórios:**
```
id-account: {account_id}
```

**Request Body:**
```json
{
  "items": [
    {
      "idProduct": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 2
    },
    {
      "idProduct": "660e8400-e29b-41d4-a716-446655440001",
      "quantity": 5
    }
  ]
}
```

**Response (201 Created):**
```json
{
  "id": "770e8400-e29b-41d4-a716-446655440002",
  "idAccount": "123e4567-e89b-12d3-a456-426614174000",
  "date": "2025-10-31T14:30:45",
  "items": [
    {
      "id": "880e8400-e29b-41d4-a716-446655440003",
      "product": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "name": "Arroz Integral",
        "price": 12.50,
        "unit": "kg"
      },
      "quantity": 2,
      "subtotal": 25.00
    },
    {
      "id": "990e8400-e29b-41d4-a716-446655440004",
      "product": {
        "id": "660e8400-e29b-41d4-a716-446655440001",
        "name": "Feijão Preto",
        "price": 8.90,
        "unit": "kg"
      },
      "quantity": 5,
      "subtotal": 44.50
    }
  ],
  "total": 69.50
}
```

---

### 2. Listar Todos os Pedidos
**Endpoint:** `GET /order`

**Descrição:** Retorna todos os pedidos da conta autenticada.

**Headers Obrigatórios:**
```
id-account: {account_id}
```

**Response (200 OK):**
```json
[
  {
    "id": "770e8400-e29b-41d4-a716-446655440002",
    "idAccount": "123e4567-e89b-12d3-a456-426614174000",
    "date": "2025-10-31T14:30:45",
    "items": [
      {
        "id": "880e8400-e29b-41d4-a716-446655440003",
        "product": {
          "id": "550e8400-e29b-41d4-a716-446655440000",
          "name": "Arroz Integral",
          "price": 12.50,
          "unit": "kg"
        },
        "quantity": 2,
        "subtotal": 25.00
      }
    ],
    "total": 25.00
  }
]
```

---

### 3. Buscar Pedido por ID
**Endpoint:** `GET /order/{id}`

**Descrição:** Retorna os detalhes de um pedido específico.

**Headers Obrigatórios:**
```
id-account: {account_id}
```

**Exemplo:** `GET /order/770e8400-e29b-41d4-a716-446655440002`

**Response (200 OK):**
```json
{
  "id": "770e8400-e29b-41d4-a716-446655440002",
  "idAccount": "123e4567-e89b-12d3-a456-426614174000",
  "date": "2025-10-31T14:30:45",
  "items": [
    {
      "id": "880e8400-e29b-41d4-a716-446655440003",
      "product": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "name": "Arroz Integral",
        "price": 12.50,
        "unit": "kg"
      },
      "quantity": 2,
      "subtotal": 25.00
    }
  ],
  "total": 25.00
}
```

**Códigos de Status:**
- `200 OK`: Pedido encontrado
- `404 Not Found`: Pedido não encontrado ou não pertence à conta

---

### 4. Deletar Pedido
**Endpoint:** `DELETE /order/{id}`

**Descrição:** Remove um pedido e todos os seus itens (cascade delete).

**Headers Obrigatórios:**
```
id-account: {account_id}
```

**Exemplo:** `DELETE /order/770e8400-e29b-41d4-a716-446655440002`

**Response (204 No Content)**

**Códigos de Status:**
- `204 No Content`: Pedido deletado com sucesso
- `404 Not Found`: Pedido não encontrado ou não pertence à conta

---

## Testes com Postman

### Criar Pedido
![POST /order](42F1E123-ADF2-438D-BDEB-5CFA237B7690_1_105_c.jpeg)

### Listar Pedidos
![GET /order](CB33D68D-B8DF-4669-84C4-9B8256F176AF.jpeg)

### Buscar Pedido por ID
![GET /order/{id}](DB161413-70B3-47FA-B809-10BE87FD6B79_1_105_c.jpeg)

### Deletar Pedido
![DELETE /order/{id}](42712892-5549-4115-B95C-47F5B3D60D4A_1_105_c.jpeg)
![DELETE /order/{id}](66342924-5351-4530-979C-2F78972EFBCA.jpeg)
---

## Estrutura do Projeto

### Order API (FeignClient)
```
📁 api/
└── 📁 order/
    ├── 📁 src/
    │   └── 📁 main/
    │       └── 📁 java/
    │           └── 📁 store/
    │               └── 📁 order/
    │                   ├── 📄 OrderController.java
    │                   ├── 📄 OrderIn.java
    │                   ├── 📄 OrderOut.java
    │                   ├── 📄 OrderItemIn.java
    │                   └── 📄 OrderItemOut.java
    ├── 📄 pom.xml
    └── 📄 README.md
```

### Order Service API
```
📁 api/
└── 📁 order_service/
    ├── 📁 src/
    │   ├── 📁 main/
    │   │   ├── 📁 java/
    │   │   │   └── 📁 store/
    │   │   │       └── 📁 order/
    │   │   │           ├── 📄 OrderApplication.java
    │   │   │           ├── 📄 OrderResource.java
    │   │   │           ├── 📄 OrderService.java
    │   │   │           ├── 📄 OrderParser.java
    │   │   │           ├── 📄 Order.java
    │   │   │           ├── 📄 OrderItem.java
    │   │   │           ├── 📄 OrderModel.java
    │   │   │           ├── 📄 OrderItemModel.java
    │   │   │           ├── 📄 OrderRepository.java
    │   │   │           └── 📄 OrderItemRepository.java
    │   │   └── 📁 resources/
    │   │       ├── 📄 application.yaml
    │   │       └── 📁 db/
    │   │           └── 📁 migration/
    │   │               ├── 📄 V2025.10.27.001__create_schema.sql
    │   │               ├── 📄 V2025.10.27.002__create_table_order.sql
    │   │               └── 📄 V2025.10.27.003__create_table_order_item.sql
    │   └── 📁 test/
    │       └── 📁 java/
    │           └── 📁 store/
    │               └── 📁 order/
    ├── 📄 Dockerfile
    └── 📄 pom.xml
```

---

## Repositórios

| Componente | Descrição | Link do Repositório |
|------------|-----------|---------------------|
| **Order API** | FeignClient para comunicação entre microserviços | https://github.com/DeenaElOrra/order |
| **Order Service** | Serviço principal com lógica de negócios e persistência | https://github.com/DeenaElOrra/order_service |
| **Product API** | Integração para buscar informações de produtos | https://github.com/DeenaElOrra/product |
| **Gateway** | API Gateway para roteamento de requisições | https://github.com/DeenaElOrra/gateway-service |

---

### Variáveis de Ambiente (Order Service)
```bash
DATABASE_URL=jdbc:postgresql://localhost:5432/store
DATABASE_USERNAME=store
DATABASE_PASSWORD=store
```

### Executar Order Service
```bash
cd api/order_service
mvn clean install
mvn spring-boot:run
```

A aplicação estará disponível em: `http://localhost:8080`

---

## Schema do Banco de Dados

### Schema: `orders`

```sql
CREATE SCHEMA IF NOT EXISTS orders;
```

### Tabela: `orders`

```sql
CREATE TABLE orders.orders (
    id_order VARCHAR(36) NOT NULL,
    id_account VARCHAR(36) NOT NULL,
    dt_created_at TIMESTAMP NOT NULL,
    vl_total DOUBLE PRECISION NOT NULL,
    CONSTRAINT pk_order PRIMARY KEY (id_order)
);
```

**Campos:**
- `id_order`: UUID do pedido (chave primária)
- `id_account`: UUID da conta do usuário
- `dt_created_at`: Data e hora de criação do pedido
- `vl_total`: Valor total do pedido

### Tabela: `order_item`

```sql
CREATE TABLE orders.order_item (
    id_order_item VARCHAR(36) NOT NULL,
    id_order VARCHAR(36) NOT NULL,
    id_product VARCHAR(36) NOT NULL,
    qt_quantity INTEGER NOT NULL,
    vl_price DOUBLE PRECISION NOT NULL,
    vl_subtotal DOUBLE PRECISION NOT NULL,
    CONSTRAINT pk_order_item PRIMARY KEY (id_order_item),
    CONSTRAINT fk_order_item_order FOREIGN KEY (id_order)
        REFERENCES orders.orders(id_order) ON DELETE CASCADE
);
```

---


## Exemplos de Uso

### Criar Pedido Completo

**Request:**
```bash
curl -X POST "http://localhost:8080/order" \
  -H "Content-Type: application/json" \
  -H "id-account: 123e4567-e89b-12d3-a456-426614174000" \
  -d '{
    "items": [
      {
        "idProduct": "550e8400-e29b-41d4-a716-446655440000",
        "quantity": 3
      },
      {
        "idProduct": "660e8400-e29b-41d4-a716-446655440001",
        "quantity": 2
      }
    ]
  }'
```

**Response:**
```json
{
  "id": "770e8400-e29b-41d4-a716-446655440002",
  "idAccount": "123e4567-e89b-12d3-a456-426614174000",
  "date": "2025-10-31T14:30:45",
  "items": [
    {
      "id": "880e8400-e29b-41d4-a716-446655440003",
      "product": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "name": "Arroz Integral",
        "price": 12.50,
        "unit": "kg"
      },
      "quantity": 3,
      "subtotal": 37.50
    },
    {
      "id": "990e8400-e29b-41d4-a716-446655440004",
      "product": {
        "id": "660e8400-e29b-41d4-a716-446655440001",
        "name": "Feijão Preto",
        "price": 8.90,
        "unit": "kg"
      },
      "quantity": 2,
      "subtotal": 17.80
    }
  ],
  "total": 55.30
}
```

---

## Logging

A aplicação utiliza logging em nível DEBUG para o pacote `store`:

```yaml
logging:
  level:
    store: debug
```

**Eventos logados:**
- Requisições HTTP recebidas
- Chamadas ao Product Service
- Operações de banco de dados
- Erros e exceções
- Cálculos de totais

---

## Dockerfile

```dockerfile
FROM openjdk:25-slim
VOLUME /tmp
COPY target/*.jar /app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

**Build e Run:**
```bash
cd api/order_service
mvn clean package
docker build -t order-service .
docker run -p 8080:8080 \
  -e DATABASE_URL=jdbc:postgresql://host:5432/store \
  -e DATABASE_USERNAME=store \
  -e DATABASE_PASSWORD=store \
  order-service
```

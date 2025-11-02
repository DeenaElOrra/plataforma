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
![POST /order](./docs/postman-create-order.png)

### Listar Pedidos
![GET /order](./docs/postman-list-orders.png)

### Buscar Pedido por ID
![GET /order/{id}](./docs/postman-get-order.png)

### Deletar Pedido
![DELETE /order/{id}](./docs/postman-delete-order.png)

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
| **Order API** | FeignClient para comunicação entre microserviços | [Link do repositório] |
| **Order Service** | Serviço principal com lógica de negócios e persistência | [Link do repositório] |
| **Product API** | Integração para buscar informações de produtos | [Link do repositório] |
| **Gateway** | API Gateway para roteamento de requisições | [Link do repositório] |

---

## Tecnologias Utilizadas

### Order API
- Spring Boot 3.5.5
- Java 21
- Spring Cloud OpenFeign 2025.0.0
- Lombok
- Product API (dependência)

### Order Service
- Spring Boot 3.5.5
- Java 21
- Spring Data JPA
- PostgreSQL 17.6
- Flyway (Database Migration)
- Spring Cloud OpenFeign
- Lombok
- Order API (dependência)
- Product API (dependência)

---

## Configuração e Execução

### Pré-requisitos
- Java 21+
- Maven 3.8+
- PostgreSQL 17.6+
- Product Service rodando (dependência externa)

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

**Campos:**
- `id_order_item`: UUID do item (chave primária)
- `id_order`: UUID do pedido (chave estrangeira)
- `id_product`: UUID do produto (referência ao Product Service)
- `qt_quantity`: Quantidade do item
- `vl_price`: Preço unitário no momento da compra
- `vl_subtotal`: Subtotal do item (quantidade × preço)

**Características:**
- **CASCADE DELETE**: Ao deletar um pedido, todos os itens são deletados automaticamente
- **Relacionamento 1:N**: Um pedido pode ter múltiplos itens

---

## Arquitetura

### Camadas da Aplicação

1. **Controller Layer** - `OrderResource`: Gerencia requisições HTTP
2. **Service Layer** - `OrderService`: Contém a lógica de negócios
3. **Repository Layer** - `OrderRepository`, `OrderItemRepository`: Acesso aos dados
4. **DTO Layer** - `OrderIn/OrderOut`, `OrderItemIn/OrderItemOut`: Contratos da API
5. **Domain Model** - `Order`, `OrderItem`: Representa as entidades de negócio
6. **Entity Model** - `OrderModel`, `OrderItemModel`: Representa as entidades do banco

### Parser/Converter Layer

**OrderParser** - Responsável por:
- Converter `OrderIn` → `Order` (domain)
- Converter `Order` → `OrderOut` (response)
- Enriquecer items com dados do Product Service
- Calcular subtotais e totais

### Comunicação entre Microserviços

O **Order Service** integra-se com o **Product Service** para enriquecer os pedidos:

```java
@FeignClient(name = "product", url = "http://product-service:80")
public interface ProductController {
    @GetMapping("/product/{id}")
    ResponseEntity<ProductOut> findProduct(@PathVariable String id);
}
```

**Fluxo de Enriquecimento:**
1. Cliente envia `OrderIn` com apenas `idProduct` e `quantity`
2. Order Service chama Product Service para cada produto
3. Obtém `ProductOut` completo (nome, preço, unidade)
4. Calcula subtotal: `quantity × product.price`
5. Calcula total do pedido: soma de todos os subtotais
6. Persiste no banco de dados
7. Retorna `OrderOut` enriquecido

---

## Multi-Tenancy baseado em Headers

### Isolamento de Dados por Conta

Todos os endpoints requerem o header `id-account` para garantir isolamento de dados:

```java
@GetMapping("/order")
public ResponseEntity<List<OrderOut>> findAll(
    @RequestHeader("id-account") String idAccount
) {
    // Retorna apenas pedidos da conta especificada
}
```

**Segurança:**
- Cada conta só pode ver seus próprios pedidos
- Validação do `id-account` em todas as operações
- Operações de busca e delete verificam propriedade

---

## Integrações

### Product Service

**URL:** `http://product-service:80`

**Endpoint Utilizado:**
- `GET /product/{id}` - Buscar informações de um produto

**Momento da Integração:**
- **Criação de pedido**: Busca produto para cada item
- **Listagem de pedidos**: Busca produto para cada item (re-enriquecimento)
- **Busca de pedido por ID**: Busca produto para cada item

**Dados Obtidos:**
```json
{
  "id": "string",
  "name": "string",
  "price": "double",
  "unit": "string"
}
```

**Tratamento de Erros:**
- Se o produto não existir, o item não é adicionado ao pedido
- Logs de erro são registrados

---

## Cálculo Automático de Valores

### Subtotal do Item

```java
subtotal = quantity × product.price
```

**Exemplo:**
- Quantidade: 5
- Preço unitário: R$ 8.90
- Subtotal: 5 × 8.90 = **R$ 44.50**

### Total do Pedido

```java
total = Σ(subtotal de todos os itens)
```

**Exemplo:**
- Item 1: R$ 25.00
- Item 2: R$ 44.50
- Total: 25.00 + 44.50 = **R$ 69.50**

---

## Repositórios Personalizados

### OrderRepository

```java
public interface OrderRepository extends CrudRepository<OrderModel, String> {
    List<OrderModel> findByIdAccount(String idAccount);
    Optional<OrderModel> findByIdAndIdAccount(String id, String idAccount);
}
```

**Métodos:**
- `findByIdAccount`: Lista todos os pedidos de uma conta
- `findByIdAndIdAccount`: Busca pedido validando propriedade

### OrderItemRepository

```java
public interface OrderItemRepository extends CrudRepository<OrderItemModel, String> {
    List<OrderItemModel> findByIdOrder(String idOrder);
    void deleteByIdOrder(String idOrder);
}
```

**Métodos:**
- `findByIdOrder`: Lista todos os itens de um pedido
- `deleteByIdOrder`: Deleta todos os itens de um pedido

---

## Validações e Tratamento de Erros

### Validações Implementadas

1. **Account ID obrigatório**: Header `id-account` deve estar presente
2. **Produto existe**: Valida se o produto existe no Product Service
3. **Quantidade válida**: Quantidade deve ser maior que zero
4. **Propriedade do pedido**: Valida se o pedido pertence à conta

### Tratamento de Erros

| Erro | Status Code | Descrição |
|------|-------------|-----------|
| Pedido não encontrado | 404 | ID do pedido inválido ou não pertence à conta |
| Produto não encontrado | 400/404 | Produto referenciado não existe |
| Account ID ausente | 400 | Header `id-account` não fornecido |
| Erro de integração | 500 | Falha ao comunicar com Product Service |

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

---

## Recursos Avançados

### 1. Transações

Operações de criação e deleção são transacionais:

```java
@Service
@Transactional
public class OrderService {
    // Métodos transacionais
}
```

### 2. UUID Generation

IDs são gerados automaticamente usando UUID:

```java
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private String id;
```

### 3. Cascade Delete

Deleção em cascata de itens ao deletar pedido:

```sql
CONSTRAINT fk_order_item_order FOREIGN KEY (id_order)
    REFERENCES orders(id_order) ON DELETE CASCADE
```

### 4. Timestamp Automático

Data de criação definida automaticamente:

```java
private LocalDateTime createdAt = LocalDateTime.now();
```

---

## Testes

### Executar Testes
```bash
cd api/order_service
mvn test
```

---

## Licença

[Adicione informações sobre a licença do projeto]

---

## Contato

[Adicione informações de contato ou contribuição]

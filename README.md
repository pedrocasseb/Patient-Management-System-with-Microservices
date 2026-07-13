# Patient Management System — Microservices

Sistema de gestão de pacientes construído como uma arquitetura de microsserviços em **Java 21 / Spring Boot**, com comunicação síncrona via **REST** e **gRPC**, mensageria assíncrona via **Kafka**, autenticação centralizada via **JWT**, e infraestrutura como código para **AWS (via LocalStack)** usando **AWS CDK**.

O projeto foi desenhado como um estudo/portfólio de padrões comuns em sistemas distribuídos: API Gateway, service-to-service auth, event-driven architecture, e provisionamento de infraestrutura em nuvem.

---

## Sumário

- [Arquitetura](#arquitetura)
- [Serviços](#serviços)
- [Stack Tecnológica](#stack-tecnológica)
- [Pré-requisitos](#pré-requisitos)
- [Como Executar](#como-executar)
  - [Build local com Maven](#build-local-com-maven)
  - [Execução via Docker](#execução-via-docker)
  - [Deploy em LocalStack via AWS CDK](#deploy-em-localstack-via-aws-cdk)
- [Autenticação](#autenticação)
- [Endpoints da API](#endpoints-da-api)
- [Fluxo de Criação de Paciente](#fluxo-de-criação-de-paciente)
- [Variáveis de Ambiente](#variáveis-de-ambiente)
- [Testes](#testes)
- [Estrutura do Projeto](#estrutura-do-projeto)
- [Documentação da API (Swagger)](#documentação-da-api-swagger)
- [Licença](#licença)

---

## Arquitetura

```
                                   ┌──────────────────────┐
                                   │        Client         │
                                   └───────────┬───────────┘
                                               │ HTTP
                                               ▼
                                   ┌──────────────────────┐
                                   │      API Gateway      │  :4004
                                   │  (Spring Cloud Gateway│
                                   │   + filtro JWT)        │
                                   └───────────┬───────────┘
                       ┌───────────────────────┼───────────────────────┐
                       │ /auth/**              │ /api/patients/**      │
                       ▼                       ▼                      │
           ┌───────────────────────┐ ┌───────────────────────┐        │
           │     Auth Service       │ │    Patient Service     │◄──────┘
           │   (login, JWT, users)  │ │ (CRUD de pacientes)     │
           │        :4005           │ │        :4000            │
           │      PostgreSQL        │ │      PostgreSQL         │
           └───────────────────────┘ └──────────┬──────────────┘
                                                  │
                          ┌───────────────────────┼───────────────────────┐
                          │ gRPC                  │ Kafka (tópico "patient")
                          ▼                       ▼
              ┌───────────────────────┐ ┌───────────────────────┐
              │    Billing Service     │ │   Analytics Service    │
              │  (contas de cobrança)  │ │  (consumo de eventos)  │
              │   :4001 (REST)         │ │        :4002           │
              │   :9001 (gRPC)         │ │                        │
              └───────────────────────┘ └───────────────────────┘
```

- **API Gateway** é o único ponto de entrada exposto publicamente; roteia requisições e valida JWT antes de repassar para os serviços internos.
- **Auth Service** emite e valida tokens JWT, e mantém a base de usuários.
- **Patient Service** gerencia o CRUD de pacientes. Ao criar um paciente, ele:
  1. chama o **Billing Service** via **gRPC** para gerar uma conta de cobrança;
  2. publica um evento `PatientEvent` (Protobuf) no tópico Kafka `patient`.
- **Analytics Service** consome os eventos do Kafka para fins de analytics/observabilidade.
- **Billing Service** expõe um serviço gRPC (`BillingService.CreateBillingAccount`).

---

## Serviços

| Serviço              | Porta REST | Porta gRPC | Banco de dados | Responsabilidade                                            |
|----------------------|:----------:|:----------:|-----------------|--------------------------------------------------------------|
| `api-gateway`        | 4004       | –          | –               | Roteamento e validação de JWT (Spring Cloud Gateway WebFlux) |
| `auth-service`       | 4005       | –          | PostgreSQL       | Login, emissão e validação de token JWT                     |
| `patient-service`     | 4000       | –          | PostgreSQL       | CRUD de pacientes, orquestra billing e eventos               |
| `billing-service`     | 4001       | 9001       | –               | Criação de contas de cobrança (servidor gRPC)                |
| `analytics-service`   | 4002       | –          | –               | Consumo de eventos de pacientes via Kafka                    |
| `infrastructure`      | –          | –          | –               | IaC (AWS CDK) para provisionar tudo no LocalStack             |
| `integration-test`    | –          | –          | –               | Testes de integração end-to-end (REST Assured)               |

---

## Stack Tecnológica

- **Java 21** / **Spring Boot 4.1**
- **Spring Cloud Gateway (WebFlux)** — API Gateway reativo
- **Spring Data JPA** + **PostgreSQL** — persistência
- **Spring Security** + **JJWT** — autenticação/autorização baseada em JWT
- **Spring Kafka** + **Protobuf** — mensageria orientada a eventos
- **gRPC** (`grpc-spring-boot-starter`) — comunicação síncrona entre `patient-service` e `billing-service`
- **springdoc-openapi** — documentação OpenAPI/Swagger
- **AWS CDK (Java)** + **LocalStack** — infraestrutura como código (VPC, RDS, MSK, ECS Fargate, ALB)
- **REST Assured** + **JUnit 5** — testes de integração
- **Maven** — build e gerenciamento de dependências
- **Docker** — empacotamento dos serviços

---

## Pré-requisitos

- JDK 21+
- Maven 3.9+ (ou use o wrapper `./mvnw` incluso em cada módulo)
- Docker (para build/execução em containers e para o LocalStack)
- [AWS CDK CLI](https://docs.aws.amazon.com/cdk/v2/guide/getting_started.html) e [LocalStack](https://docs.localstack.cloud/) (apenas para o fluxo de deploy simulado na AWS)

---

## Como Executar

### Build local com Maven

Cada serviço é um módulo Maven independente (não há um POM pai agregador). Para compilar/testar um serviço:

```bash
cd auth-service      # ou patient-service, billing-service, analytics-service, api-gateway
./mvnw clean install
```

Para rodar um serviço localmente:

```bash
./mvnw spring-boot:run
```

> **Ordem sugerida de subida local:** `auth-service` → `patient-service` → `billing-service` → `analytics-service` → `api-gateway`, já que o gateway depende dos demais estarem no ar para rotear corretamente, e o `patient-service` depende do `billing-service` (gRPC) e de um broker Kafka disponível.

Como o `api-gateway` resolve os hosts dos outros serviços pelo nome (`auth-service`, `patient-service` — ver `application.yml`), ao rodar tudo fora de containers/Docker Compose você deve ajustar essas URIs para `localhost` ou exportar as variáveis de ambiente equivalentes (ver [Variáveis de Ambiente](#variáveis-de-ambiente)).

### Execução via Docker

Cada serviço possui seu próprio `Dockerfile` (multi-stage: build com Maven + runtime `eclipse-temurin:21-jre`). Para buildar a imagem de um serviço:

```bash
docker build -t patient-service ./patient-service
```

Repita para `auth-service`, `billing-service`, `analytics-service` e `api-gateway`. Em seguida, suba os containers em uma rede Docker comum, garantindo que os nomes dos containers batam com os hosts usados no `application.yml` do gateway (`auth-service`, `patient-service`), e providencie um PostgreSQL para `auth-service`/`patient-service` e um broker Kafka para `patient-service`/`analytics-service`.

> Não há um `docker-compose.yml` no repositório — o provisionamento completo (incluindo bancos, MSK/Kafka e rede) é feito via CDK/LocalStack, descrito abaixo.

### Deploy em LocalStack via AWS CDK

O módulo `infrastructure` define uma stack CDK (`com.pm.stack.LocalStack`) que provisiona, contra um LocalStack local:

- uma **VPC** dedicada;
- duas instâncias **RDS PostgreSQL** (`auth-service-db`, `patient-service-db`) com health checks Route53;
- um cluster **MSK (Kafka)**;
- um **cluster ECS Fargate** com um serviço por microsserviço (`auth-service`, `billing-service`, `analytics-service`, `patient-service`);
- um **Application Load Balancer** na frente do `api-gateway` (perfil `prod`).

Para sintetizar/gerar o template:

```bash
cd infrastructure
./mvnw clean install
cdk synth --app "mvn -e -q compile exec:java -Dexec.mainClass=com.pm.stack.LocalStack.main"
```

O `cdk.out/` gerado pode então ser aplicado contra uma instância do LocalStack em execução (`localstack start`), usando `cdklocal deploy` no lugar de `cdk deploy`.

---

## Autenticação

O fluxo de autenticação é centralizado no `auth-service` e validado pelo `api-gateway`:

1. O cliente faz `POST /auth/login` com e-mail e senha.
2. O `auth-service` valida as credenciais (BCrypt) e retorna um **JWT** assinado.
3. Nas chamadas subsequentes, o cliente envia o header `Authorization: Bearer <token>`.
4. O `api-gateway` aplica o filtro customizado `JwtValidationGatewayFilterFactory` nas rotas protegidas (ex.: `/api/patients/**`), que chama `GET /validate` no `auth-service` antes de liberar o roteamento.
5. Requisições sem token válido recebem `401 Unauthorized` diretamente do gateway, sem sequer chegar ao serviço de destino.

A rota `/auth/**` (login) **não** passa pelo filtro de JWT, já que é o próprio endpoint que emite o token.

---

## Endpoints da API

Todas as chamadas abaixo passam pelo **API Gateway** (`http://localhost:4004`). Exemplos completos em formato `.http` estão disponíveis em [`api-requests/`](./api-requests) e [`grpc-requests/`](./grpc-requests).

### Auth Service (`/auth`)

| Método | Rota             | Descrição                          | Autenticado |
|--------|------------------|-------------------------------------|:-----------:|
| POST   | `/auth/login`    | Autentica e retorna um JWT           | Não         |
| GET    | `/auth/validate` | Valida um JWT (uso interno/gateway)  | Sim         |

**Exemplo — login:**

```http
POST http://localhost:4004/auth/login
Content-Type: application/json

{
  "email": "testuser@test.com",
  "password": "password123"
}
```

### Patient Service (`/api/patients`)

| Método | Rota                 | Descrição                    | Autenticado |
|--------|----------------------|-------------------------------|:-----------:|
| GET    | `/api/patients`      | Lista todos os pacientes       | Sim         |
| POST   | `/api/patients`      | Cria um novo paciente          | Sim         |
| PUT    | `/api/patients/{id}` | Atualiza um paciente existente | Sim         |
| DELETE | `/api/patients/{id}` | Remove um paciente             | Sim         |

**Exemplo — criar paciente:**

```http
POST http://localhost:4004/api/patients
Content-Type: application/json
Authorization: Bearer {{token}}

{
  "name": "Kafka test",
  "email": "kafka_test@example.com",
  "address": "123 main street",
  "dateOfBirth": "1995-09-09",
  "registeredDate": "2024-11-28"
}
```

### Billing Service (gRPC)

Serviço `BillingService` (`billing_service.proto`), consumido internamente pelo `patient-service`:

```
rpc CreateBillingAccount (BillingRequest) returns (BillingResponse);
```

---

## Fluxo de Criação de Paciente

Ao criar um paciente (`POST /api/patients`), o `patient-service`:

1. Valida duplicidade de e-mail e persiste o paciente no PostgreSQL.
2. Chama `BillingServiceGrpcClient.createBillingAccount(...)` via **gRPC** para abrir uma conta de cobrança no `billing-service`.
3. Publica um evento `PatientEvent` (Protobuf, serializado como `byte[]`) no tópico Kafka **`patient`**.
4. O `analytics-service` consome esse evento (`@KafkaListener(topics = "patient", groupId = "analytics-service")`) para fins de observabilidade/analytics.

Esse fluxo combina comunicação **síncrona** (gRPC, no caminho crítico da requisição) com **assíncrona** (Kafka, fire-and-forget) — um padrão comum para desacoplar efeitos colaterais não bloqueantes do fluxo principal.

---

## Variáveis de Ambiente

| Variável                          | Serviço          | Descrição                                              | Exemplo                                  |
|-----------------------------------|------------------|----------------------------------------------------------|-------------------------------------------|
| `AUTH_SERVICE_URL`                 | `api-gateway`     | Base URL usada pelo filtro de validação de JWT            | `http://auth-service:4005`                |
| `SPRING_PROFILES_ACTIVE`           | `api-gateway`     | Ativa o profile `prod` (rotas via `host.docker.internal`) | `prod`                                    |
| `SPRING_DATASOURCE_URL`            | `auth-service`, `patient-service` | Conexão JDBC com o PostgreSQL              | `jdbc:postgresql://localhost:5432/auth-service-db` |
| `SPRING_DATASOURCE_USERNAME`       | `auth-service`, `patient-service` | Usuário do banco                            | `admin_user`                              |
| `SPRING_DATASOURCE_PASSWORD`       | `auth-service`, `patient-service` | Senha do banco                              | –                                         |
| `SPRING_KAFKA_BOOTSTRAP_SERVERS`   | `patient-service`, `analytics-service` | Endereços do cluster Kafka             | `localhost:9092`                          |
| `BILLING_SERVICE_ADDRESS`          | `patient-service` | Host do servidor gRPC do billing                          | `localhost`                               |
| `BILLING_SERVICE_GRPC_PORT`        | `patient-service` | Porta do servidor gRPC do billing                         | `9001`                                    |
| `JWT_SECRET`                       | `auth-service`    | Segredo usado para assinar/validar os JWTs                | –                                         |

> Ao rodar `api-gateway` fora do profile `prod` (default) sem Docker/ECS, defina `AUTH_SERVICE_URL` manualmente (via `-D` ou variável de ambiente) — o Spring resolve `${auth.service.url}` a partir dessa variável por *relaxed binding*.

---

## Testes

Cada serviço possui sua própria suíte de testes unitários (`src/test/java`), executável com:

```bash
./mvnw test
```

O módulo [`integration-test`](./integration-test) contém testes **end-to-end** com REST Assured, que exercitam a stack completa através do gateway (`http://localhost:4004`):

- `AuthIntegrationTest` — login com credenciais válidas/inválidas.
- `PatientIntegrationTest` — login seguido de listagem de pacientes autenticada.

Para rodá-los, a stack completa (gateway, auth-service, patient-service e dependências) precisa estar em execução:

```bash
cd integration-test
./mvnw test
```

---

## Estrutura do Projeto

```
patient-management/
├── api-gateway/          # Spring Cloud Gateway + filtro de validação JWT
├── auth-service/         # Login, emissão/validação de JWT, usuários
├── patient-service/       # CRUD de pacientes, cliente gRPC, produtor Kafka
├── billing-service/       # Servidor gRPC de contas de cobrança
├── analytics-service/     # Consumidor Kafka de eventos de pacientes
├── infrastructure/        # AWS CDK (Java) para provisionamento no LocalStack
├── integration-test/      # Testes end-to-end (REST Assured + JUnit 5)
├── api-requests/          # Exemplos de requisições HTTP (.http)
└── grpc-requests/         # Exemplos de chamadas gRPC (.http)
```

---

## Documentação da API (Swagger)

`auth-service` e `patient-service` expõem documentação OpenAPI via `springdoc-openapi`:

- Auth Service: `http://localhost:4005/swagger-ui.html`
- Patient Service: `http://localhost:4000/swagger-ui.html`

Através do gateway, os specs também ficam disponíveis em:

- `http://localhost:4004/api-docs/auth`
- `http://localhost:4004/api-docs/patients`

---

## Licença

Projeto pessoal para fins de estudo/portfólio. Nenhuma licença foi definida até o momento.

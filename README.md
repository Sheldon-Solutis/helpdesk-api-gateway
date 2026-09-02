# HelpDesk Platform

Plataforma de HelpDesk baseada em microsserviços: gestão de usuários, chamados
de suporte e notificações assíncronas via RabbitMQ, com um API Gateway como
único ponto de entrada para o frontend em React.

## Arquitetura

```
                         ┌─────────────┐
   React (5173) ───────► │   Gateway   │  (8080, Spring Cloud Gateway)
                         └──────┬──────┘
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
       user-service      ticket-service    notification-service
         (8081)              (8082)               (8083)
              │                 │                     ▲
              │                 │   RabbitMQ events    │
              │                 └──────────────────────┘
              ▼                 ▼                     ▼
          user_db           ticket_db          notification_db
        (PostgreSQL)       (PostgreSQL)          (PostgreSQL)
```

- O frontend nunca chama os microsserviços diretamente — tudo passa pelo
  gateway em `/api/users/**`, `/api/tickets/**` e `/api/notifications/**`.
- `ticket-service` valida `customerId`/`technicianId` chamando `user-service`
  diretamente (HTTP interno, fora do gateway) e publica eventos no RabbitMQ.
- `notification-service` consome esses eventos e grava o histórico de
  notificações.
- Cada serviço tem seu próprio banco PostgreSQL — nenhum serviço acessa o
  banco de outro.

## Como executar

Pré-requisitos: Docker e Docker Compose.

Este `docker-compose.yml` (na raiz) builda tudo localmente a partir do
código-fonte — é o caminho mais simples para testar o projeto como está,
neste monorepo. Se você já dividiu os serviços em repositórios separados e
está publicando imagens no GHCR, use `helpdesk-infra/docker-compose.yml` em
vez deste (ele só puxa as imagens já publicadas — veja
[`POLYREPO.md`](./POLYREPO.md)).

```bash
docker compose up --build
```

Isso sobe, nesta ordem (respeitando os healthchecks):

| Serviço              | Porta | URL                                    |
|----------------------|-------|----------------------------------------|
| Frontend (React)     | 5173  | http://localhost:5173                  |
| API Gateway          | 8080  | http://localhost:8080/api              |
| user-service         | 8081  | http://localhost:8081/swagger-ui.html  |
| ticket-service       | 8082  | http://localhost:8082/swagger-ui.html  |
| notification-service | 8083  | http://localhost:8083/swagger-ui.html  |
| PostgreSQL           | 5432  | ———                                      |
| RabbitMQ (management)| 15672 | http://localhost:15672                |

Para derrubar tudo (mantendo os volumes): `docker compose down`.
Para apagar também os dados: `docker compose down -v`.

## Fluxo de eventos (RabbitMQ)

`ticket-service` publica no exchange `ticket.events` (topic):

| Evento                   | Routing key            | Quando                                |
|---------------------------|------------------------|----------------------------------------|
| `TicketCreatedEvent`      | `ticket.created`       | Ao abrir um chamado                    |
| `TicketAssignedEvent`     | `ticket.assigned`      | Ao atribuir/trocar o técnico           |
| `TicketStatusChangedEvent`| `ticket.status.changed`| Ao transicionar o status               |

`notification-service` declara uma fila durável por evento, consome, e grava
uma `Notification` (histórico consultável em `GET /api/notifications`).

## Variáveis de ambiente

Cada microsserviço Java lê exclusivamente de variáveis de ambiente (sem
valores hardcoded), com defaults locais úteis para rodar fora do Docker:

- `DATABASE_URL`, `DATABASE_USER`, `DATABASE_PASSWORD`
- `RABBITMQ_HOST`, `RABBITMQ_PORT`, `RABBITMQ_USER`, `RABBITMQ_PASSWORD` (ticket-service e notification-service)
- `USER_SERVICE_URL` (ticket-service e gateway-service)
- `TICKET_SERVICE_URL`, `NOTIFICATION_SERVICE_URL` (gateway-service)

O frontend lê `VITE_API_BASE_URL` (passada como build-arg no Dockerfile, já
que o Vite embute variáveis de ambiente em tempo de build).

## Rodando sem Docker (desenvolvimento local)

Backend (em cada pasta de serviço):

```bash
mvn spring-boot:run
```

(requer um PostgreSQL e um RabbitMQ locais, ou ajuste as variáveis de ambiente
para apontar para instâncias já existentes)

Frontend:

```bash
cd frontend
npm install
cp .env.example .env   # ajuste VITE_API_BASE_URL se necessário
npm run dev
```

## Testes

```bash
cd user-service && mvn test
cd ticket-service && mvn test
cd notification-service && mvn test
```
## Documentação da API

Cada microsserviço expõe Swagger UI em `/swagger-ui.html` e o contrato OpenAPI
em `/v3/api-docs`. Todas as chamadas do frontend passam pelo gateway, então em
produção a rota efetiva é `/api/<recurso>/...` na porta 8080.

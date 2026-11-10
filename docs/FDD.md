# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Feature** | Webhooks outbound de mudança de status de pedido |
| **Status** | Pronto para revisão técnica (Bruno, Diego e Larissa) |
| **Data da reunião de origem** | Não informada na transcrição (quinta-feira, 09:00) |
| **Proposta de origem** | [RFC-001](RFC.md) |
| **Decisões** | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) · [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) · [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) · [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) · [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) · [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) · [ADR-007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md) |
| **Rastreabilidade** | Cada item com ID (`FDD-*`) tem linha no [TRACKER](TRACKER.md) |

> Este documento responde **como construir**. O porquê de cada decisão está nos ADRs, e o problema de negócio está no [PRD](PRD.md). O FDD não repete as justificativas.

---

## 1. Contexto e motivação técnica

O OMS (Express 4 + Prisma 5 + MySQL 8) muda o status de pedidos em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de `prisma.$transaction(async (tx) => …)`. As transições válidas (FDD-CTX-01) ficam em `src/modules/orders/order.status.ts`:

```
PENDING → PAID | CANCELLED
PAID → PROCESSING | CANCELLED
PROCESSING → SHIPPED | CANCELLED
SHIPPED → DELIVERED
DELIVERED, CANCELLED → (terminais)
```

Não existe nenhum mecanismo de evento, fila ou notificação externa. Os clientes B2B consultam `GET /api/v1/orders` repetidamente para detectar mudanças ([09:00] Marcos).

A motivação técnica tem três partes:

- tirar o polling do cliente, com entrega abaixo de 10 s ([09:02] Marcos);
- não pôr I/O externo dentro da transação de `changeStatus` ([09:04] Bruno);
- não perder eventos: status mudou ⇒ evento existe ([09:40] Bruno).

## 2. Objetivos técnicos

| ID | Objetivo | Critério verificável |
|---|---|---|
| FDD-OBJ-01 | Atomicidade entre mudança de status e registro do evento | Um rollback de `changeStatus` não deixa linha em `webhook_outbox`. Uma falha no insert da outbox faz rollback do status ([09:40] Bruno) |
| FDD-OBJ-02 | Alvo de latência abaixo de 10 s para endpoint saudável | Medir `delivered_at - created_at`; perfil de carga, concorrência e critério estatístico precisam ser definidos antes de declarar o alvo validado (RFC-OQ-09) ([09:02] Marcos, [09:09] Diego) |
| FDD-OBJ-03 | Nenhum evento descartado silenciosamente | Todo evento termina `DELIVERED` ou tem linha em `webhook_dead_letter` ([09:15] Diego) |
| FDD-OBJ-04 | Entregas verificáveis pelo cliente | 100% das requisições com `X-Signature` HMAC-SHA256 válido para a secret do endpoint ([09:20] Sofia) |
| FDD-OBJ-05 | Zero dependência nova e zero mudança de contrato nas rotas existentes | `package.json` sem novas `dependencies`; `tests/orders.test.ts` passa sem alteração ([09:29] Bruno) |

## 3. Escopo e exclusões

### 3.1 Em escopo

| ID | Item |
|---|---|
| FDD-ESC-01 | Módulo `src/modules/webhooks/` com CRUD de configuração, rotação de secret, histórico de entregas e replay de DLQ ([09:27] Bruno) |
| FDD-ESC-02 | Tabelas `webhooks`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter` (migração Prisma aditiva) |
| FDD-ESC-03 | Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada por `changeStatus` ([09:41] Bruno) |
| FDD-ESC-04 | Entry point `src/worker.ts` + script `npm run worker` ([09:11] Larissa) |
| FDD-ESC-05 | Evento único `order.status_changed` ([09:43] Diego) |

### 3.2 Exclusões (não implementar nesta fase)

| ID | Exclusão | Fonte |
|---|---|---|
| FDD-EXC-01 | Webhooks inbound (clientes enviando para nós) | [09:02] Marcos |
| FDD-EXC-02 | Notificação por e-mail quando o webhook falha | [09:37] Larissa |
| FDD-EXC-03 | Rate limiting de saída por cliente (observar antes) | [09:39] Larissa |
| FDD-EXC-04 | Dashboard visual para o cliente | [09:40] Larissa |
| FDD-EXC-05 | Múltiplos workers / particionamento por `order_id` | [09:13] Diego |
| FDD-EXC-06 | Arquivamento de linhas entregues da outbox (~30 dias) | [09:08] Diego |
| FDD-EXC-07 | Evento na **criação** do pedido. `OrderService.create` grava o `PENDING` inicial sem passar por `changeStatus`, e nenhuma transição leva a `PENDING` (`src/modules/orders/order.status.ts`) | `src/modules/orders/order.service.ts` |
| FDD-EXC-08 | Garantia de ordenação global e exactly-once | [09:13] Larissa, [09:25] Diego |

## 4. Arquitetura de componentes

```
src/
├── server.ts                        (existente) API HTTP
├── worker.ts                        (NOVO) entry point do worker
└── modules/webhooks/                (NOVO)
    ├── webhook.routes.ts            buildWebhookRouter, buildCustomerWebhookRouter, buildWebhookAdminRouter
    ├── webhook.controller.ts        WebhookController (handlers Express)
    ├── webhook.service.ts           WebhookService (CRUD, rotação, deliveries, replay)
    ├── webhook.repository.ts        WebhookRepository (Prisma)
    ├── webhook.schemas.ts           schemas Zod
    ├── webhook.publisher.ts         publishWebhookEvent(tx, …)
    ├── webhook.worker.ts            WebhookWorker (loop, claim, envio, retry, DLQ)
    ├── webhook.signer.ts            generateSecret(), sign(body, secrets)
    └── webhook.constants.ts         BACKOFF_SCHEDULE_MS, retry limit (pending), MAX_PAYLOAD_BYTES…
```

| ID | Constante (`webhook.constants.ts`) | Valor | Fonte |
|---|---|---|---|
| FDD-CONST-01 | `BACKOFF_SCHEDULE_MS` | `[60_000, 300_000, 1_800_000, 7_200_000, 43_200_000]` (1m, 5m, 30m, 2h, 12h) | [09:17] Larissa |
| FDD-CONST-02 | Limite de tentativas | **Pendente:** não fixar `MAX_RETRIES` nem total de chamadas antes de resolver [RFC-OQ-08](RFC.md#5-questões-em-aberto) | [09:16] Diego, [09:17] Diego, [09:17] Larissa |
| FDD-CONST-03 | `MAX_PAYLOAD_BYTES` | `65_536` (64 KB) | [09:24] Larissa |
| FDD-CONST-04 | `SECRET_GRACE_PERIOD_MS` | `86_400_000` (24 h) | [09:21] Sofia |
| FDD-CONST-05 | `EVENT_TYPE_ORDER_STATUS_CHANGED` | `'order.status_changed'` | [09:43] Diego |
| FDD-CONST-06 | `SUBSCRIBABLE_STATUSES` | Status que são destino de alguma transição: `PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED`. Derivado de `canTransition` | `src/modules/orders/order.status.ts` |

| ID | Variável de ambiente (`src/config/env.ts`) | Default | Fonte |
|---|---|---|---|
| FDD-ENV-01 | `WEBHOOK_POLL_INTERVAL_MS` | `2000` | [09:09] Diego |
| FDD-ENV-02 | `WEBHOOK_HTTP_TIMEOUT_MS` | `10000` | [09:42] Diego |
| FDD-ENV-03 | `WEBHOOK_BATCH_SIZE` | `10` (valor inicial para "batch pequeno", ajustável) | [09:08] Diego |

## 5. Modelo de dados

Acréscimos a `prisma/schema.prisma`. **FDD-DADOS-08:** todas as PKs são UUID `Char(36)` ([09:51] Larissa), e timestamps usam `DATETIME(3)`, como nas tabelas existentes (`prisma/migrations/20260519182739_init/migration.sql`).

```prisma
enum WebhookOutboxStatus {
  PENDING      // pendente
  PROCESSING   // processando (reivindicado pelo worker)
  DELIVERED    // entregue
  FAILED       // falhou definitivamente (cópia na DLQ)
}

model Webhook {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(128)
  previousSecret          String?   @db.VarChar(128)
  previousSecretExpiresAt DateTime?
  events                  Json      // OrderStatus[] — ex.: ["SHIPPED","DELIVERED"]
  active                  Boolean   @default(true)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer    Customer            @relation(fields: [customerId], references: [id])
  outbox      WebhookOutbox[]
  deliveries  WebhookDelivery[]
  deadLetters WebhookDeadLetter[]

  @@index([customerId, active])
  @@map("webhooks")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36)
  eventId       String              @db.Char(36)
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36) // sem FK: o snapshot sobrevive a DELETE /orders/:id
  eventType     String              @db.VarChar(64)
  payload       String              @db.MediumText // JSON serializado; bytes exatos que são assinados
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastError     String?             @db.VarChar(500)
  deliveredAt   DateTime?
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  webhook     Webhook             @relation(fields: [webhookId], references: [id])
  deliveries  WebhookDelivery[]
  deadLetters WebhookDeadLetter[]

  @@unique([eventId, webhookId])
  @@index([status, nextAttemptAt])
  @@index([createdAt])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id             String   @id @default(uuid()) @db.Char(36)
  outboxId       String   @db.Char(36)
  webhookId      String   @db.Char(36)
  eventId        String   @db.Char(36)
  attempt        Int
  success        Boolean
  responseStatus Int?
  responseBody   String?  @db.Text // truncado em 64 KB
  errorCode      String?  @db.VarChar(64)
  durationMs     Int
  createdAt      DateTime @default(now())

  outbox  WebhookOutbox @relation(fields: [outboxId], references: [id])
  webhook Webhook       @relation(fields: [webhookId], references: [id])

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id           String    @id @default(uuid()) @db.Char(36)
  outboxId     String    @db.Char(36)
  webhookId    String    @db.Char(36)
  eventId      String    @db.Char(36)
  payload      String    @db.MediumText
  reason       String    @db.VarChar(64)   // código WEBHOOK_*
  lastError    String?   @db.VarChar(500)
  attempts     Int
  failedAt     DateTime  @default(now())
  replayedAt   DateTime?
  replayedById String?   @db.Char(36)

  outbox  WebhookOutbox @relation(fields: [outboxId], references: [id])
  webhook Webhook       @relation(fields: [webhookId], references: [id])

  @@index([webhookId, failedAt])
  @@map("webhook_dead_letter")
}

// Em model Customer (existente), adicionar a relação inversa:
//   webhooks Webhook[]
```

> **Ação referencial pendente:** as relações acima mostram apenas os campos/FKs. A política de exclusão (hard delete, soft delete e efeitos sobre outbox, deliveries e DLQ) não foi decidida; não tratar a ausência de `onDelete` neste exemplo como escolha aprovada. Resolver RFC-OQ-12 antes de gerar a migração.

| ID | Regra de modelagem | Fonte |
|---|---|---|
| FDD-DADOS-01 | `webhooks` guarda url, secret, `customer_id` e estado ativo | [09:21] Bruno |
| FDD-DADOS-02 | `webhook_outbox` tem **uma linha por (evento × webhook inscrito)**. Todas as linhas de uma mesma mudança de status compartilham o `eventId` ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)), e cada endpoint tem retry independente | [09:25] Diego |
| FDD-DADOS-03 | Os índices da outbox cobrem status e `created_at` | [09:08] Diego |
| FDD-DADOS-04 | O payload é gravado já serializado (snapshot), o que garante bytes estáveis para o HMAC | [09:52] Larissa |
| FDD-DADOS-05 | A DLQ é uma tabela separada com payload, motivo e timestamp. Um mesmo `outboxId` pode ter mais de uma linha se falhar de novo depois de um replay | [09:18] Diego |
| FDD-DADOS-06 | `webhook_deliveries` registra **cada tentativa** (sucesso/falha, resposta, duração) e alimenta `GET /webhooks/:id/deliveries` | [09:34] Marcos |
| FDD-DADOS-07 | `url` e `secret` são lidas **no momento do envio**, não no snapshot. Uma edição de URL ou rotação de secret vale também para eventos pendentes. Só o payload é imutável | [09:21] Sofia |

## 6. Contratos públicos

**FDD-CONTRATO-10 — Convenções:** base path `/api/v1` (`src/app.ts`). Todas as rotas exigem `Authorization: Bearer <JWT>` (`authenticate`). Formato de erro: `{ "error": { "code", "message", "details?" } }` (`src/middlewares/error.middleware.ts`). Listas: `{ "data": [...], "pagination": { page, pageSize, total, totalPages } }` (`src/shared/http/response.ts`).

**FDD-CONTRATO-09 — Autorização:** qualquer role autenticada (`ADMIN` ou `OPERATOR`) no CRUD, na rotação e no histórico ([09:37] Sofia). Só `ADMIN` no replay de DLQ ([09:36] Sofia). **FDD-CONTRATO-00:** o `customerId` vai **no body (criação) ou no path (listagem)**, as duas opções admitidas na reunião; ele **não é derivado do JWT**, porque o token é de um usuário operador ([09:32] Larissa, [09:32] Bruno). A relação de ownership/escopo do usuário sobre o customer não foi decidida; `authenticate`/role isoladamente não define autorização entre customers. Resolver RFC-OQ-10 antes de implementar essas rotas.

### FDD-CONTRATO-01 — `POST /api/v1/webhooks` (criar webhook)

Fonte: [09:31] Marcos. A secret é gerada pela plataforma e **devolvida só nesta resposta e na rotação**.

**Request**
```http
POST /api/v1/webhooks
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{
  "customerId": "5b0c7a4e-2f1d-4c8e-9a51-0f3e2d7c9b10",
  "url": "https://integracoes.atlascomercial.com.br/oms/webhooks",
  "events": ["SHIPPED", "DELIVERED"]
}
```

**Response `201 Created`**
```json
{
  "id": "0d9b3f8e-6a2c-4b71-8f0e-2c5d9a1e7b34",
  "customerId": "5b0c7a4e-2f1d-4c8e-9a51-0f3e2d7c9b10",
  "url": "https://integracoes.atlascomercial.com.br/oms/webhooks",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "3f6c1e0b9a7d4e2f8c5b1a0d9e7f6c3b2a1d0e9f8c7b6a5d4e3f2c1b0a9d8e7f",
  "previousSecretExpiresAt": null,
  "createdAt": "2026-11-03T13:02:11.482Z",
  "updatedAt": "2026-11-03T13:02:11.482Z"
}
```

| Status | Quando |
|---|---|
| `201` | Criado |
| `400 VALIDATION_ERROR` | `url` não é `https` (`details[].message` começa com `WEBHOOK_INVALID_URL`), `events` vazio ou com status não inscrevível (`WEBHOOK_INVALID_EVENTS`), `customerId` não é UUID |
| `401 UNAUTHORIZED` | Token ausente ou inválido |
| `404 WEBHOOK_CUSTOMER_NOT_FOUND` | `customerId` inexistente |

### FDD-CONTRATO-02 — `GET /api/v1/customers/:customerId/webhooks` (listar webhooks de um customer)

Fonte: [09:33] Bruno. **Nunca devolve `secret`.** O router do módulo é montado em `/customers/:customerId/webhooks` (`Router({ mergeParams: true })`), registrado **antes** de `/customers` em `src/routes/index.ts`, para que a rota não passe pelo router de customers.

**Request**
```http
GET /api/v1/customers/5b0c7a4e-2f1d-4c8e-9a51-0f3e2d7c9b10/webhooks?page=1&pageSize=20
Authorization: Bearer eyJhbGciOi...
```

**Response `200 OK`**
```json
{
  "data": [
    {
      "id": "0d9b3f8e-6a2c-4b71-8f0e-2c5d9a1e7b34",
      "customerId": "5b0c7a4e-2f1d-4c8e-9a51-0f3e2d7c9b10",
      "url": "https://integracoes.atlascomercial.com.br/oms/webhooks",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "previousSecretExpiresAt": null,
      "createdAt": "2026-11-03T13:02:11.482Z",
      "updatedAt": "2026-11-03T13:02:11.482Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

| Status | Quando |
|---|---|
| `200` | Lista (pode vir vazia) |
| `400 VALIDATION_ERROR` | `customerId` não é UUID; `pageSize` > 100 |
| `401 UNAUTHORIZED` | Token ausente ou inválido |
| `404 WEBHOOK_CUSTOMER_NOT_FOUND` | `customerId` inexistente |
| `401 UNAUTHORIZED` | Token ausente ou inválido |

### FDD-CONTRATO-03 — `PATCH /api/v1/webhooks/:id` (editar webhook)

Fonte: [09:33] Bruno. Campos editáveis: `url`, `events`, `active`, com pelo menos um obrigatório. `secret` e `customerId` **não** são editáveis por aqui e são ignorados se enviados.

**Request**
```http
PATCH /api/v1/webhooks/0d9b3f8e-6a2c-4b71-8f0e-2c5d9a1e7b34
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{ "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"], "active": true }
```

**Response `200 OK`**
```json
{
  "id": "0d9b3f8e-6a2c-4b71-8f0e-2c5d9a1e7b34",
  "customerId": "5b0c7a4e-2f1d-4c8e-9a51-0f3e2d7c9b10",
  "url": "https://integracoes.atlascomercial.com.br/oms/webhooks",
  "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true,
  "previousSecretExpiresAt": null,
  "createdAt": "2026-11-03T13:02:11.482Z",
  "updatedAt": "2026-11-04T09:15:40.010Z"
}
```

| Status | Quando |
|---|---|
| `200` | Atualizado. A mudança de `events` só vale para eventos futuros ([ADR-007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md)) |
| `400 VALIDATION_ERROR` | Body vazio, `url` não `https` (`WEBHOOK_INVALID_URL`), `events` inválido (`WEBHOOK_INVALID_EVENTS`) |
| `401 UNAUTHORIZED` | Token ausente ou inválido |
| `404 WEBHOOK_NOT_FOUND` | `id` inexistente |

### FDD-CONTRATO-04 — `DELETE /api/v1/webhooks/:id` (remover webhook)

Fonte: [09:33] Bruno. A reunião pede operação de remoção, mas não decide hard delete versus desativação lógica nem o efeito sobre outbox pendente, histórico de deliveries e DLQ. O endpoint e a resposta `204` abaixo são proposta de contrato; seu efeito persistente fica pendente em RFC-OQ-12. Não implementar cascata até essa decisão.

**Request**
```http
DELETE /api/v1/webhooks/0d9b3f8e-6a2c-4b71-8f0e-2c5d9a1e7b34
Authorization: Bearer eyJhbGciOi...
```

**Response `204 No Content`** (sem corpo)

| Status | Quando |
|---|---|
| `204` | Removido |
| `401 UNAUTHORIZED` | Token ausente ou inválido |
| `404 WEBHOOK_NOT_FOUND` | `id` inexistente |

### FDD-CONTRATO-05 — `POST /api/v1/webhooks/:id/rotate-secret` (rotacionar secret)

Fonte: [09:21] Sofia. Gera uma nova secret. A atual vira `previousSecret`, válida por 24 h. Se já existia uma `previousSecret` em grace, ela é descartada na hora.

**Request**
```http
POST /api/v1/webhooks/0d9b3f8e-6a2c-4b71-8f0e-2c5d9a1e7b34/rotate-secret
Authorization: Bearer eyJhbGciOi...
```

**Response `200 OK`**
```json
{
  "id": "0d9b3f8e-6a2c-4b71-8f0e-2c5d9a1e7b34",
  "secret": "9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f1a0b9c8d7e6f5a4b3c2d1e0f9a8b",
  "previousSecretExpiresAt": "2026-11-05T10:00:00.000Z"
}
```

| Status | Quando |
|---|---|
| `200` | Rotacionada |
| `401 UNAUTHORIZED` | Token ausente ou inválido |
| `404 WEBHOOK_NOT_FOUND` | `id` inexistente |

### FDD-CONTRATO-06 — `GET /api/v1/webhooks/:id/deliveries` (histórico de entregas)

Fonte: [09:34] Marcos. Ordenado do mais recente para o mais antigo, paginado com `pageSize` máximo de 100 (os "últimos 100").

**Request**
```http
GET /api/v1/webhooks/0d9b3f8e-6a2c-4b71-8f0e-2c5d9a1e7b34/deliveries?page=1&pageSize=100
Authorization: Bearer eyJhbGciOi...
```

**Response `200 OK`**
```json
{
  "data": [
    {
      "id": "b4a1c9d2-3e5f-4a6b-8c7d-9e0f1a2b3c4d",
      "eventId": "7c1e4b2a-9d3f-4e8a-b6c5-1f2e3d4c5b6a",
      "attempt": 2,
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "errorCode": null,
      "durationMs": 184,
      "payload": {
        "event_id": "7c1e4b2a-9d3f-4e8a-b6c5-1f2e3d4c5b6a",
        "event_type": "order.status_changed",
        "timestamp": "2026-11-04T14:03:12.345Z",
        "order_id": "e2f1a0b9-c8d7-4e6f-a5b4-c3d2e1f0a9b8",
        "order_number": "ORD-000123",
        "from_status": "PROCESSING",
        "to_status": "SHIPPED",
        "customer_id": "5b0c7a4e-2f1d-4c8e-9a51-0f3e2d7c9b10",
        "total_cents": 158000
      },
      "createdAt": "2026-11-04T14:04:14.901Z"
    },
    {
      "id": "a3b2c1d0-e9f8-4a7b-9c6d-5e4f3a2b1c0d",
      "eventId": "7c1e4b2a-9d3f-4e8a-b6c5-1f2e3d4c5b6a",
      "attempt": 1,
      "success": false,
      "responseStatus": null,
      "responseBody": null,
      "errorCode": "WEBHOOK_DELIVERY_TIMEOUT",
      "durationMs": 10002,
      "payload": { "event_id": "7c1e4b2a-9d3f-4e8a-b6c5-1f2e3d4c5b6a", "...": "..." },
      "createdAt": "2026-11-04T14:03:14.120Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 2, "totalPages": 1 }
}
```

| Status | Quando |
|---|---|
| `200` | Lista (pode vir vazia) |
| `400 VALIDATION_ERROR` | `id` não é UUID; `pageSize` > 100 |
| `401 UNAUTHORIZED` | Token ausente ou inválido |
| `404 WEBHOOK_NOT_FOUND` | `id` inexistente |

### FDD-CONTRATO-07 — `POST /api/v1/admin/webhooks/dead-letter/:id/replay` (replay de DLQ)

Fonte: [09:18] Diego, [09:36] Sofia. Exige `ADMIN`. Recoloca o evento na outbox como `PENDING`, com o **mesmo `eventId`**, e registra quem fez o replay.

**Request**
```http
POST /api/v1/admin/webhooks/dead-letter/c9d8e7f6-a5b4-4c3d-8e2f-1a0b9c8d7e6f/replay
Authorization: Bearer eyJhbGciOi...   (role ADMIN)
```

**Response `202 Accepted`**
```json
{
  "deadLetterId": "c9d8e7f6-a5b4-4c3d-8e2f-1a0b9c8d7e6f",
  "outboxId": "f1e2d3c4-b5a6-4978-8a6b-5c4d3e2f1a0b",
  "eventId": "7c1e4b2a-9d3f-4e8a-b6c5-1f2e3d4c5b6a",
  "webhookId": "0d9b3f8e-6a2c-4b71-8f0e-2c5d9a1e7b34",
  "status": "PENDING",
  "replayedAt": "2026-11-05T08:30:00.000Z",
  "replayedById": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d"
}
```

| Status | Quando |
|---|---|
| `202` | Reenfileirado. A entrega acontece de forma assíncrona |
| `401 UNAUTHORIZED` | Token ausente ou inválido |
| `403 FORBIDDEN` | Role diferente de `ADMIN` (`requireRole('ADMIN')`) |
| `404 WEBHOOK_DEAD_LETTER_NOT_FOUND` | `id` inexistente |
| `409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | Essa linha da DLQ já foi reprocessada |
| `422 WEBHOOK_INACTIVE` | O webhook de destino está `active: false`. É preciso reativar via PATCH antes |

### FDD-CONTRATO-08 — Requisição de entrega (outbound, do worker para o cliente)

Fontes: payload [09:43] Diego; headers [09:44] Diego e [09:44] Sofia; HMAC [09:22] Sofia.

```http
POST https://integracoes.atlascomercial.com.br/oms/webhooks
Content-Type: application/json
X-Event-Id: 7c1e4b2a-9d3f-4e8a-b6c5-1f2e3d4c5b6a
X-Webhook-Id: 0d9b3f8e-6a2c-4b71-8f0e-2c5d9a1e7b34
X-Timestamp: 2026-11-04T14:03:14.118Z
X-Signature: sha256=5d41402abc4b2a76b9719d911017c592ae2f0e8c6f1a3b4c5d6e7f8091a2b3c4

{"event_id":"7c1e4b2a-9d3f-4e8a-b6c5-1f2e3d4c5b6a","event_type":"order.status_changed","timestamp":"2026-11-04T14:03:12.345Z","order_id":"e2f1a0b9-c8d7-4e6f-a5b4-c3d2e1f0a9b8","order_number":"ORD-000123","from_status":"PROCESSING","to_status":"SHIPPED","customer_id":"5b0c7a4e-2f1d-4c8e-9a51-0f3e2d7c9b10","total_cents":158000}
```

| Elemento | Semântica |
|---|---|
| `event_id` / `X-Event-Id` | UUID gerado na inserção da outbox, igual em retries e replays. Chave de deduplicação do cliente |
| `event_type` | Sempre `order.status_changed` nesta fase |
| `timestamp` (corpo) | ISO 8601 do momento da mudança de status (snapshot) |
| `X-Timestamp` (header) | ISO 8601 do **envio** daquela tentativa, para detecção de replay attack pelo cliente, se ele quiser |
| `order_*`, `from_status`, `to_status`, `customer_id`, `total_cents` | Snapshot do pedido, **sem `items`** (detalhes via `GET /api/v1/orders/:id`) |
| `X-Webhook-Id` | ID do cadastro que originou o envio |
| `X-Signature` | `sha256=` + hex de `HMAC_SHA256(secret, corpo_bruto)`. **Durante o grace period** de rotação: `sha256=<hex_nova>,sha256=<hex_anterior>`. O formato está pendente de validação na revisão de segurança (RFC-OQ-07) |
| Resposta esperada | Qualquer `2xx` em até 10 s = entregue. Qualquer outra coisa = falha com retry |

**Verificação do lado do cliente (referência para o portal do desenvolvedor):**

```ts
import { createHmac, timingSafeEqual } from 'node:crypto';

export function isValidSignature(rawBody: string, header: string, secret: string): boolean {
  const expected = createHmac('sha256', secret).update(rawBody, 'utf8').digest('hex');
  return header.split(',').some((part) => {
    const received = part.trim().replace(/^sha256=/, '');
    return received.length === expected.length &&
      timingSafeEqual(Buffer.from(received), Buffer.from(expected));
  });
}
```

### Schemas Zod (`src/modules/webhooks/webhook.schemas.ts`)

```ts
import { z } from 'zod';
import { OrderStatus } from '@prisma/client';
import { canTransition } from '../orders/order.status.js';

const ALL = Object.values(OrderStatus);
export const SUBSCRIBABLE_STATUSES = ALL.filter((to) => ALL.some((from) => canTransition(from, to)));

const httpsUrl = z.string().url().max(2048)
  .refine((u) => new URL(u).protocol === 'https:', { message: 'WEBHOOK_INVALID_URL: url must use https' });

const events = z.array(z.nativeEnum(OrderStatus)).min(1)
  .refine((list) => list.every((s) => SUBSCRIBABLE_STATUSES.includes(s)), {
    message: 'WEBHOOK_INVALID_EVENTS: PENDING is not a subscribable status',
  })
  .transform((list) => Array.from(new Set(list)));

export const webhookIdParamSchema = z.object({ id: z.string().uuid() });
export const deadLetterIdParamSchema = z.object({ id: z.string().uuid() });
export const createWebhookSchema = z.object({ customerId: z.string().uuid(), url: httpsUrl, events });
export const updateWebhookSchema = z
  .object({ url: httpsUrl.optional(), events: events.optional(), active: z.boolean().optional() })
  .refine((b) => Object.keys(b).length > 0, { message: 'at least one field is required' });
export const customerIdParamSchema = z.object({ customerId: z.string().uuid() });
export const paginationQuerySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  pageSize: z.coerce.number().int().min(1).max(100).default(20),
});
// listagem de webhooks: params customerIdParamSchema + query paginationQuerySchema
// histórico de entregas: params webhookIdParamSchema + query paginationQuerySchema
```

## 7. Fluxos detalhados

### 7.1 FDD-FLUXO-01 — Criação do evento na outbox (dentro de `changeStatus`)

```mermaid
sequenceDiagram
  participant C as OrderController
  participant S as OrderService.changeStatus
  participant P as publishWebhookEvent
  participant DB as MySQL (tx)
  C->>S: changeStatus(id, {toStatus}, userId)
  S->>DB: BEGIN / findUnique(order + items)
  S->>S: canTransition(from, to)
  S->>DB: debitStock / replenishStock (se aplicável)
  S->>DB: update orders.status
  S->>DB: insert order_status_history
  S->>P: publishWebhookEvent(tx, order, from, to)
  P->>DB: select webhooks where customerId=order.customerId and active
  P->>P: filtra events ∋ to; se vazio → return 0
  P->>P: eventId = uuidv4(); payload = JSON.stringify(snapshot)
  P->>DB: createMany webhook_outbox (1 linha por webhook)
  S->>DB: findUnique(refreshed) / COMMIT
  Note over S,DB: qualquer erro em P → exceção → ROLLBACK de tudo
```

Passo a passo:

1. `changeStatus` executa exatamente como hoje até o `tx.orderStatusHistory.create`.
2. Chama `await publishWebhookEvent(tx, order, from, to)` ([09:41] Bruno).
3. `publishWebhookEvent` busca `tx.webhook.findMany({ where: { customerId: order.customerId, active: true } })` e filtra em memória os que têm `to` em `events`. O JSON não é indexável, e o volume por customer é pequeno ([09:34] Bruno).
4. Se nenhum webhook passar no filtro, retorna `0` sem inserir ([09:34] Bruno).
5. Gera **um** `eventId` (`uuidv4`) para a mudança de status ([09:25] Diego).
6. Monta o snapshot `{ event_id, event_type, timestamp: new Date().toISOString(), order_id, order_number, from_status, to_status, customer_id, total_cents }` e serializa uma vez ([09:43] Diego, [09:52] Larissa).
7. `tx.webhookOutbox.createMany` com uma linha por webhook: `status PENDING`, `attempts 0`, `nextAttemptAt now`.
8. Loga `webhook_event_staged` em `debug` (pré-commit) e retorna a quantidade.
9. Se o passo 3 ou o 7 lançar erro, a exceção sobe, o Prisma faz rollback, o status **não** muda e o cliente da API recebe `500 INTERNAL_SERVER_ERROR` ([09:40] Bruno).

### 7.2 FDD-FLUXO-02 — Processamento pelo worker

```mermaid
flowchart TD
  A[start: recoverStuck PROCESSING→PENDING] --> B[tick]
  B --> C[select PENDING where nextAttemptAt<=now order by createdAt limit BATCH]
  C -->|vazio| W[aguarda 2s] --> B
  C --> D[updateMany → PROCESSING]
  D --> E{para cada evento, em ordem}
  E --> F[carrega webhook]
  F -->|active=false| DLQ1[DLQ reason WEBHOOK_INACTIVE]
  F -->|secret vazia| DLQ2[DLQ reason WEBHOOK_SECRET_REQUIRED]
  F --> G{bytes(payload) > 64KB?}
  G -->|sim| DLQ3[DLQ reason WEBHOOK_PAYLOAD_TOO_LARGE]
  G -->|não| H[assina + POST com timeout 10s, redirect manual]
  H --> I[grava webhook_deliveries]
  I -->|2xx| J[DELIVERED, deliveredAt=now]
  I -->|falha| K[Fluxo de retry 7.3]
  J --> E
  K --> E
  E -->|fim do lote| W
```

1. **Bootstrap (`src/worker.ts`):** cria o `PrismaClient` com `createPrismaClient()`, instancia `WebhookWorker` com `logger.child({ component: 'webhook-worker' })` e registra `SIGINT`/`SIGTERM`, no mesmo formato de `src/server.ts` ([09:11] Larissa, [09:30] Bruno).
2. **Recuperação:** na inicialização, `updateMany({ where: { status: PROCESSING }, data: { status: PENDING } })`. Com worker único, qualquer `PROCESSING` na partida é resto de crash. O reenvio é aceitável porque a garantia é at-least-once ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)).
3. **Loop:** `setTimeout` encadeado, não `setInterval`, para ciclos não se sobreporem. O próximo tick começa 2 s **depois do fim** do anterior ([09:09] Diego).
4. **Claim:** busca até `WEBHOOK_BATCH_SIZE` IDs `PENDING` com `nextAttemptAt <= now`, em ordem de `createdAt ASC`, e marca `PROCESSING` com `updateMany` condicionado a `status = PENDING` ([09:08] Diego, [09:12] Diego).
5. **Processamento sequencial** dentro do lote, na ordem de `createdAt`, o que preserva a ordem por `order_id` no caminho feliz ([09:12] Diego).
6. **Pré-checagens** (não geram chamada HTTP e vão direto para a DLQ, sem retry, porque retry não resolveria):
   - webhook `active = false` → `WEBHOOK_INACTIVE`;
   - secret ausente → `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno);
   - `Buffer.byteLength(payload) > 65_536` → `WEBHOOK_PAYLOAD_TOO_LARGE` ([09:23] Sofia, [09:24] Larissa).
7. **Assinatura:** `X-Signature` com a secret atual e, se `previousSecretExpiresAt > now`, também com a anterior ([09:21] Sofia).
8. **Envio:** `fetch(url, { method: 'POST', body: payload, headers, signal: AbortSignal.timeout(10_000), redirect: 'manual' })`. Um redirect `3xx` conta como falha, para não seguir para destino não `https` nem não cadastrado ([09:23] Sofia).
9. **Registro da tentativa:** em uma transação, insere `webhook_deliveries` (tentativa, sucesso, status, corpo truncado em 64 KB, `durationMs`, `errorCode`) e atualiza a linha da outbox ([09:34] Marcos).
10. **Sucesso (`2xx`):** `status DELIVERED`, `attempts += 1`, `deliveredAt = now`, e log `webhook_delivery_succeeded`.
11. **Falha:** fluxo 7.3.
12. **Erro inesperado no tick** (ex.: banco indisponível): loga `webhook_worker_tick_failed` e segue para o próximo tick. Linhas que ficaram `PROCESSING` são recuperadas no próximo boot.
13. **Shutdown:** para de agendar ticks, termina o evento em andamento (até 10 s), executa `prisma.$disconnect()` e sai com `0`.

### 7.3 FDD-FLUXO-03 — Retry com backoff

Os cinco intervalos abaixo foram citados na reunião. A contagem total de chamadas é ambígua: se forem cinco chamadas totais, nem todos os intervalos seriam usados; se forem cinco retentativas, há um envio inicial adicional. Não fixar a semântica de `attempts`, limite nem transição à DLQ até resolver [RFC-OQ-08](RFC.md#5-questões-em-aberto).

| Ordem do intervalo listado | Delay discutido após falha | Acumulado dos delays listados |
|---|---|---|---|
| 1 | 1 min | 1 min |
| 2 | 5 min | 6 min |
| 3 | 30 min | 36 min |
| 4 | 2 h | 2 h 36 min |
| 5 | 12 h | 14 h 36 min |

Fonte: [09:17] Diego, [09:17] Larissa.

Pseudocódigo de integração deve aguardar a resolução RFC-OQ-08 para definir qual índice da sequência usar, quantas chamadas executar e em qual falha mover para DLQ. O código de produção não deve inferir esses limites a partir do comprimento do array.

Falhas que disparam retry ([09:42] Diego):

| Causa | `errorCode` da tentativa |
|---|---|
| Sem resposta em 10 s | `WEBHOOK_DELIVERY_TIMEOUT` |
| DNS, conexão recusada, TLS | `WEBHOOK_DELIVERY_NETWORK_ERROR` |
| Resposta fora de `2xx` (inclui `3xx`) | `WEBHOOK_DELIVERY_HTTP_ERROR` |

### 7.4 FDD-FLUXO-04 — DLQ e replay

**Entrada na DLQ** (`moveToDeadLetter`, em uma transação):
1. `webhook_outbox.status = FAILED`, `attempts`, `lastError`.
2. Insert em `webhook_dead_letter` com `outboxId`, `webhookId`, `eventId`, `payload` (cópia), `reason` (`WEBHOOK_*`), `lastError`, `attempts` e `failedAt` ([09:18] Diego).

**Replay** (`WebhookService.replayDeadLetter(deadLetterId, adminUserId)`, em uma transação):
1. Busca a linha da DLQ. Se não existir, `WebhookDeadLetterNotFoundError` (404).
2. Se `replayedAt != null`, `WebhookDeadLetterAlreadyReplayedError` (409).
3. Busca o webhook. Se `active = false`, `WebhookInactiveError` (422).
4. Atualiza a outbox (`outboxId`): `status PENDING`, `attempts 0`, `nextAttemptAt now`, `lastError null`. O `eventId` se mantém ([09:18] Diego).
5. Marca a DLQ com `replayedAt = now` e `replayedById = req.user.id`.
6. Depois do commit, loga `info` `webhook_dead_letter_replayed` com `{ deadLetterId, outboxId, eventId, webhookId, replayedById }` ([09:36] Sofia).
7. Se falhar de novo, o ciclo recomeça conforme o limite de chamadas que vier a ser aprovado em RFC-OQ-08; não assumir seis chamadas.

### 7.5 FDD-FLUXO-05 — Criação de webhook e rotação de secret

1. **Criação:** valida o body (Zod) e verifica se o customer existe (`WEBHOOK_CUSTOMER_NOT_FOUND`). Gera a secret com `randomBytes(32).toString('hex')` (`node:crypto`), persiste e devolve com a secret ([09:31] Marcos).
2. **Rotação:** em uma transação, `previousSecret = secret`, `previousSecretExpiresAt = now + 24h`, `secret = nova`. Loga `webhook_secret_rotated` com `{ webhookId, userId }`, **sem valores** ([09:21] Sofia).
3. **Expiração:** o worker ignora `previousSecret` quando `previousSecretExpiresAt <= now`. A limpeza física do campo acontece na próxima rotação.

## 8. Matriz de erros previstos

### 8.1 Erros HTTP da API (novas classes em `src/shared/errors/http-errors.ts`)

| ID | Código | HTTP | Classe | Base | Quando | Fonte |
|---|---|---|---|---|---|---|
| FDD-ERR-01 | `WEBHOOK_NOT_FOUND` | 404 | `WebhookNotFoundError` | `AppError` | `:id` de webhook inexistente (PATCH, DELETE, rotate, deliveries) | [09:28] Bruno |
| FDD-ERR-02 | `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `WebhookCustomerNotFoundError` | `AppError` | `customerId` inexistente na criação ou na listagem | [09:29] Larissa |
| FDD-ERR-03 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `WebhookDeadLetterNotFoundError` | `AppError` | `:id` de DLQ inexistente | [09:18] Diego |
| FDD-ERR-04 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | `WebhookDeadLetterAlreadyReplayedError` | `ConflictError` | Replay repetido da mesma linha | [09:36] Sofia |
| FDD-ERR-05 | `WEBHOOK_INACTIVE` | 422 | `WebhookInactiveError` | `UnprocessableEntityError` | Replay para webhook desativado | [09:21] Bruno |

> As classes 404 estendem `AppError` diretamente porque `NotFoundError` fixa `code = 'NOT_FOUND'` (`src/shared/errors/http-errors.ts`).

### 8.2 Erros de validação (via `validate()`, código de topo `VALIDATION_ERROR`)

| ID | Código em `details[].message` | HTTP | Quando | Fonte |
|---|---|---|---|---|
| FDD-ERR-06 | `WEBHOOK_INVALID_URL` | 400 | `url` malformada ou com esquema diferente de `https` | [09:23] Sofia, [09:28] Bruno |
| FDD-ERR-07 | `WEBHOOK_INVALID_EVENTS` | 400 | `events` vazio, com status inexistente ou com `PENDING` | [09:33] Marcos |

Exemplo de resposta:
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [{ "path": "url", "message": "WEBHOOK_INVALID_URL: url must use https" }]
  }
}
```

### 8.3 Erros internos do worker (gravados em `webhook_deliveries.errorCode` / `webhook_dead_letter.reason` e nos logs; nunca expostos como HTTP)

| ID | Código | Tratamento | Retry? | Fonte |
|---|---|---|---|---|
| FDD-ERR-08 | `WEBHOOK_DELIVERY_TIMEOUT` | Falha da tentativa | Sim | [09:42] Diego |
| FDD-ERR-09 | `WEBHOOK_DELIVERY_NETWORK_ERROR` | Falha da tentativa | Sim | [09:14] Larissa |
| FDD-ERR-10 | `WEBHOOK_DELIVERY_HTTP_ERROR` | Falha da tentativa (status fora de 2xx) | Sim | [09:15] Diego |
| FDD-ERR-11 | `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | Vai para a DLQ | Não | [09:15] Diego |
| FDD-ERR-12 | `WEBHOOK_PAYLOAD_TOO_LARGE` | Vai para a DLQ sem chamada HTTP | Não | [09:23] Sofia |
| FDD-ERR-13 | `WEBHOOK_SECRET_REQUIRED` | Vai para a DLQ sem chamada HTTP (não se assina sem secret) | Não | [09:28] Bruno |
| FDD-ERR-14 | `WEBHOOK_INACTIVE` | Vai para a DLQ sem chamada HTTP | Não | [09:21] Bruno |

**FDD-ERR-15 —** Erros **genéricos existentes reutilizados sem alteração**: `UNAUTHORIZED` (401), `FORBIDDEN` (403 no replay sem ADMIN), `INTERNAL_SERVER_ERROR` (500, inclusive falha da outbox dentro de `changeStatus`).

## 9. Estratégias de resiliência

| ID | Estratégia | Detalhe | Fonte |
|---|---|---|---|
| FDD-RES-01 | **Timeout** | `AbortSignal.timeout(10_000)` por chamada. Estourou, é falha com retry | [09:42] Diego |
| FDD-RES-02 | **Retry com backoff fixo** | 1m/5m/30m/2h/12h via `nextAttemptAt`, sem `sleep` bloqueante, então o worker segue com outros eventos | [09:17] Larissa |
| FDD-RES-03 | **Teto e DLQ** | Após o limite de chamadas a decidir (RFC-OQ-08), mover para `FAILED` + `webhook_dead_letter`; replay manual por ADMIN | [09:15] Diego, [09:18] Diego |
| FDD-RES-04 | **Falha rápida sem retry** | Payload > 64 KB, webhook inativo e ausência de secret vão direto à DLQ | [09:23] Sofia |
| FDD-RES-05 | **Atomicidade como fallback de consistência** | Falha da outbox faz rollback do status: nunca há status sem evento | [09:41] Diego |
| FDD-RES-06 | **Recuperação de crash** | `PROCESSING → PENDING` no boot do worker. Pode reenviar, e o cliente deduplica por `X-Event-Id` | [09:24] Diego |
| FDD-RES-07 | **Isolamento de processo** | O worker cai sem afetar a API e vice-versa. Eventos acumulam na outbox e são drenados na volta | [09:11] Diego |
| FDD-RES-08 | **Ticks sem sobreposição** | `setTimeout` encadeado. Um tick lento nunca dispara outro em paralelo, o que preserva o single-worker | [09:12] Diego |
| FDD-RES-09 | **Redirect manual** | `redirect: 'manual'`; `3xx` é falha. Isso não restringe por si só destinos privados/internos ou DNS rebinding; política de destino pendente em RFC-OQ-11 | [09:23] Sofia |
| FDD-RES-10 | **Tolerância a banco indisponível** | Um erro no tick é logado e o loop continua no próximo intervalo; o worker não pode "perder" entregas por falha transitória | [09:11] Diego |

## 10. Observabilidade

**FDD-OBS-20:** o projeto não tem biblioteca de métricas nem de tracing (`package.json`), e a decisão é não adicionar dependências ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)). Logs estruturados Pino são a fonte primária. Métricas são emitidas como eventos de log com campos numéricos (agregáveis pela plataforma de logs) ou derivadas por SQL das tabelas do módulo. Tracing é feito por IDs de correlação.

### 10.1 Logs (Pino, nomes em `snake_case`, como `server_started` e `http_request`)

| ID | Evento | Nível | Onde | Campos |
|---|---|---|---|---|
| FDD-OBS-01 | `webhook_event_staged` | debug | publisher (pré-commit) | `eventId, orderId, fromStatus, toStatus, webhookCount` |
| FDD-OBS-02 | `webhook_worker_started` / `webhook_worker_stopped` | info | worker | `pollIntervalMs, batchSize, recoveredCount` |
| FDD-OBS-03 | `webhook_delivery_succeeded` | info | worker | `eventId, webhookId, orderId, attempt, statusCode, durationMs` |
| FDD-OBS-04 | `webhook_delivery_retry_scheduled` | warn | worker | `eventId, webhookId, orderId, attempt, errorCode, statusCode?, nextAttemptAt` |
| FDD-OBS-05 | `webhook_moved_to_dead_letter` | error | worker | `eventId, webhookId, orderId, attempts, reason` |
| FDD-OBS-06 | `webhook_dead_letter_replayed` | info | API | `deadLetterId, outboxId, eventId, webhookId, replayedById` (auditoria, [09:36] Sofia) |
| FDD-OBS-07 | `webhook_secret_rotated` | info | API | `webhookId, userId` (nunca o valor da secret) |
| FDD-OBS-08 | `webhook_worker_tick_failed` | error | worker | `err` |
| FDD-OBS-09 | **Redaction:** acrescentar `'*.secret'` e `'*.previousSecret'` a `redactPaths` | — | `src/shared/logger/index.ts` | Motivo: vazamento de secret em log já ocorreu ([09:22] Diego) |

### 10.2 Métricas

| ID | Métrica | Tipo | Origem | Uso / alerta sugerido |
|---|---|---|---|---|
| FDD-OBS-10 | `webhook_delivery_latency_ms` (`deliveredAt - createdAt`) | histograma | campo no log `webhook_delivery_succeeded` | Avaliar o alvo < 10 s após definir perfil de carga e critério estatístico (RFC-OQ-09; [09:02] Marcos) |
| FDD-OBS-11 | `webhook_delivery_attempts_total{result=success\|failure, errorCode}` | contador | logs 03/04/05 | Taxa de falha por webhook, base para a decisão de rate limiting (RFC-OQ-01) |
| FDD-OBS-12 | `webhook_delivery_duration_ms` | histograma | `durationMs` | Identificar endpoints lentos (head-of-line) |
| FDD-OBS-13 | `webhook_outbox_pending_total` | gauge | `SELECT COUNT(*) … WHERE status='PENDING'`, emitido no log `webhook_worker_stats` a cada 30 ticks (~1 min) | Crescimento contínuo indica worker parado ou lento |
| FDD-OBS-14 | `webhook_outbox_oldest_pending_age_seconds` | gauge | `MIN(createdAt)` dos `PENDING` com `nextAttemptAt <= now`, emitido em `webhook_worker_stats` | Acima de 10 s de forma sustentada: gatilho do RFC-OQ-03 (escala) |
| FDD-OBS-15 | `webhook_dead_letter_open_total` | gauge | `COUNT(*) WHERE replayedAt IS NULL`, emitido em `webhook_worker_stats` | Alerta para ADMIN avaliar replay; dado para a decisão sobre e-mail (RFC-OQ-02) |

### 10.3 Tracing (correlação)

| ID | Elo | Como |
|---|---|---|
| FDD-OBS-16 | Requisição da API → pedido | `X-Request-Id` / `requestId` já gerados por `src/middlewares/request-logger.middleware.ts`, com `path` contendo o `orderId` |
| FDD-OBS-17 | Pedido → evento | `webhook_event_staged` e linhas da outbox com `orderId` + `eventId` |
| FDD-OBS-18 | Evento → tentativas | `eventId` + `webhookId` em todos os logs do worker e em `webhook_deliveries` |
| FDD-OBS-19 | Nós → cliente | `X-Event-Id` e `X-Webhook-Id` enviados. O cliente reporta problemas citando esses IDs |

Tracing distribuído (OpenTelemetry) está fora do escopo: não existe na stack e exigiria dependência nova.

## 11. Integração com o sistema existente

| ID | Arquivo real | Estado atual | Integração |
|---|---|---|---|
| FDD-INT-01 | `src/modules/orders/order.service.ts` | `changeStatus` abre `$transaction`, valida, mexe no estoque, faz update em `orders` e insert em `order_status_history` | Depois de `tx.orderStatusHistory.create(...)` e antes do `findUnique` de retorno, acrescentar `await publishWebhookEvent(tx, order, from, to);`. O `TxClient` (`Prisma.TransactionClient`) já está tipado no arquivo. `create` **não** chama o publisher (FDD-EXC-07). O retorno de `changeStatus` não muda |
| FDD-INT-02 | `src/modules/orders/order.status.ts` | Tabela `transitions` e `canTransition` | **Sem alteração.** `webhook.schemas.ts` importa `canTransition` para derivar `SUBSCRIBABLE_STATUSES` (exclui `PENDING`) |
| FDD-INT-03 | `src/shared/errors/http-errors.ts` | `AppError` e subclasses como `InsufficientStockError` e `InvalidStatusTransitionError` com códigos próprios | Acrescentar `WebhookNotFoundError`, `WebhookCustomerNotFoundError` e `WebhookDeadLetterNotFoundError` (extends `AppError`, 404), `WebhookDeadLetterAlreadyReplayedError` (extends `ConflictError`) e `WebhookInactiveError` (extends `UnprocessableEntityError`) |
| FDD-INT-04 | `src/shared/errors/index.ts` | Barrel de exportação dos erros | Exportar as 5 novas classes |
| FDD-INT-05 | `src/shared/errors/app-error.ts` | `AppError(message, statusCode, errorCode, details)` | **Sem alteração.** Base das novas classes |
| FDD-INT-06 | `src/middlewares/error.middleware.ts` | Serializa `AppError`, `ZodError`, Prisma `P2002`/`P2025`; o resto vira 500 | **Sem alteração.** Os novos erros já são tratados. Como `P2003` (FK) não é mapeado, o service valida a existência do customer antes do insert para não virar 500 |
| FDD-INT-07 | `src/middlewares/auth.middleware.ts` | `authenticate`, `requireRole(...roles)`, `req.user` | `buildWebhookRouter` usa `router.use(authenticate)`. `buildWebhookAdminRouter` usa `authenticate` + `requireRole('ADMIN')`. `req.user.id` alimenta `replayedById` e os logs de auditoria |
| FDD-INT-08 | `src/middlewares/validate.middleware.ts` | `validate({ body, query, params })`; erro Zod vira `ValidationError` (`VALIDATION_ERROR`) | Usado em todas as rotas do módulo. É por causa dele que `WEBHOOK_INVALID_URL` e `WEBHOOK_INVALID_EVENTS` aparecem em `details[].message` (§8.2) |
| FDD-INT-09 | `src/shared/logger/index.ts` | Pino com `redactPaths` e `base.service` | Acrescentar `'*.secret'` e `'*.previousSecret'` a `redactPaths`. O worker usa `logger.child({ component: 'webhook-worker' })` |
| FDD-INT-10 | `src/shared/http/response.ts` | `paginated(data, page, pageSize, total)` | Usado na listagem de webhooks e no histórico de entregas |
| FDD-INT-11 | `src/config/database.ts` | `createPrismaClient()` e singleton `prisma` | `src/worker.ts` chama `createPrismaClient()` para ter a própria instância ([09:30] Bruno) |
| FDD-INT-12 | `src/config/env.ts` | `envSchema` Zod com `process.exit(1)` se inválido | Acrescentar `WEBHOOK_POLL_INTERVAL_MS`, `WEBHOOK_HTTP_TIMEOUT_MS` e `WEBHOOK_BATCH_SIZE`, todos com default, então o `.env` atual continua válido |
| FDD-INT-13 | `src/server.ts` | Bootstrap + shutdown por `SIGINT`/`SIGTERM` | Modelo de estrutura para `src/worker.ts` (bootstrap, `logger.fatal` em falha, `$disconnect` no shutdown) |
| FDD-INT-14 | `src/app.ts` | `buildControllers(prisma)` instancia repository → service → controller | Acrescentar `WebhookRepository`, `WebhookService` e `WebhookController`, e retornar `webhooks` em `Controllers` |
| FDD-INT-15 | `src/routes/index.ts` | `buildApiRouter` monta `/auth`, `/users`, `/customers`, `/products`, `/orders` | Acrescentar `webhooks: WebhookController` ao tipo `Controllers` e montar `router.use('/customers/:customerId/webhooks', buildCustomerWebhookRouter(...))` (antes de `/customers`), `router.use('/webhooks', buildWebhookRouter(...))` e `router.use('/admin/webhooks', buildWebhookAdminRouter(...))` |
| FDD-INT-16 | `prisma/schema.prisma` | Modelos `Customer`, `Order`, `OrderStatusHistory`… com UUID `Char(36)` | Acrescentar o enum e os 4 modelos do §5, mais `webhooks Webhook[]` em `Customer`. Gerar a migração com `npm run db:migrate` (aditiva, não altera tabelas existentes) |
| FDD-INT-17 | `package.json` | Scripts `dev`, `start`, `build`… | Acrescentar `"worker": "tsx watch --env-file=.env src/worker.ts"` e `"worker:start": "node --env-file=.env dist/worker.js"`. `tsconfig.build.json` já inclui `src/**/*.ts`, então `dist/worker.js` é gerado pelo `build` existente |
| FDD-INT-18 | `tests/setup.ts` | `beforeEach` apaga tabelas em ordem de FK | Apagar `webhookDeadLetter`, `webhookDelivery`, `webhookOutbox` e `webhook` **antes** de `customer.deleteMany()` |
| FDD-INT-19 | `tests/helpers/factories.ts` | `createTestCustomer`, `bootstrapAuthenticatedUser(role)` | Acrescentar `createTestWebhook(customerId, overrides)`. Usar `bootstrapAuthenticatedUser('ADMIN')` nos testes de replay |
| FDD-INT-20 | `src/middlewares/request-logger.middleware.ts` | Usa `v4 as uuidv4` do pacote `uuid` | Mesmo import para gerar `eventId` no publisher (sem nova dependência) |

> **FDD-INT-21 — autorização por customer pendente:** `authenticate` e `requireRole` confirmam identidade/role, mas a reunião e o código consultado não definem como verificar vínculo entre `req.user` e `customerId`. Não considerar o middleware proposto suficiente para isolamento multi-tenant até RFC-OQ-10 ser decidido.

## 12. Dependências e compatibilidade

| ID | Item | Detalhe |
|---|---|---|
| FDD-DEP-01 | **Runtime** | Node ≥ 20 (`engines` em `package.json`), com `fetch` global, `AbortSignal.timeout` e `node:crypto` |
| FDD-DEP-02 | **Bibliotecas** | Só as existentes: `@prisma/client` 5.22.0, `express` 4.21.1, `zod` 3.23.8, `pino` 9.5.0 e `uuid` 11.0.3. **Nenhuma nova** ([09:29] Bruno) |
| FDD-DEP-03 | **Banco** | MySQL 8.0 (`docker-compose.yml`), mesma `DATABASE_URL` para API e worker ([09:11] Diego) |
| FDD-DEP-04 | **Compatibilidade da API** | Nenhuma rota ou resposta existente muda. `PATCH /orders/:id/status` mantém contrato e status codes. Sem webhooks cadastrados, o único efeito é um `SELECT` a mais na transação |
| FDD-DEP-05 | **Compatibilidade de dados** | Migração só aditiva (4 tabelas, 1 enum, 1 relação inversa). Rollback = `DROP` das tabelas novas |
| FDD-DEP-06 | **Ordem de deploy** | 1) migração, 2) API, 3) worker (`npm run worker:start`). Eventos gerados antes do worker subir ficam `PENDING` e são drenados depois |
| FDD-DEP-07 | **Dependências externas** | Endpoint `https` do cliente, verificação de HMAC e deduplicação do lado dele. Documentação no portal do desenvolvedor, a cargo do PM ([09:26] Marcos, [09:40] Marcos) |
| FDD-DEP-08 | **Dependência de processo** | Revisão de segurança da Sofia (≥ 2 dias úteis) sobre `webhook.signer.ts` e a geração de secret antes do deploy ([09:46] Sofia) |

## 13. Critérios de aceite técnicos

| ID | Critério | Como verificar |
|---|---|---|
| FDD-AC-01 | Uma mudança de status com webhook inscrito no `to_status` cria exatamente 1 linha `PENDING` por webhook, com `eventId` compartilhado | Teste de integração (vitest + supertest), consultando `webhook_outbox` |
| FDD-AC-02 | Uma mudança de status sem webhook inscrito (ou inscrito em outros status) **não** cria linha | Teste de integração |
| FDD-AC-03 | Se o insert na outbox falhar, `orders.status` e `order_status_history` não mudam | Teste com publisher forçado a lançar erro |
| FDD-AC-04 | `POST /webhooks` com `http://` retorna 400 e `details[].message` contendo `WEBHOOK_INVALID_URL` | Teste de integração |
| FDD-AC-05 | `POST /webhooks` retorna a `secret`; `GET /webhooks` e `PATCH` nunca retornam `secret` | Teste de integração |
| FDD-AC-06 | O worker envia os headers `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp`, `X-Signature` e `Content-Type: application/json`, e a assinatura confere com `HMAC_SHA256(secret, body)` | Teste com servidor HTTP local de captura |
| FDD-AC-07 | Depois da rotação, durante 24 h, `X-Signature` confere com a secret nova **e** com a anterior; depois de 24 h, só com a nova | Teste unitário do signer com relógio controlado |
| FDD-AC-08 | Um endpoint que não responde em 10 s gera `WEBHOOK_DELIVERY_TIMEOUT`; o próximo intervalo é o primeiro da política confirmada | Teste com servidor que atrasa a resposta; confirmar política em RFC-OQ-08 |
| FDD-AC-09 | Limite de chamadas e transição para DLQ correspondem à política aprovada, sem fixar total enquanto RFC-OQ-08 estiver aberta | Critério bloqueado até decisão; depois, teste unitário do worker com relógio controlado |
| FDD-AC-10 | Payload > 64 KB vai para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE` **sem** chamada HTTP | Teste unitário |
| FDD-AC-11 | Replay por `OPERATOR` retorna 403; por `ADMIN`, retorna 202, a outbox volta a `PENDING` com o mesmo `eventId` e o log `webhook_dead_letter_replayed` contém `replayedById` | Teste de integração |
| FDD-AC-12 | Um segundo replay da mesma linha da DLQ retorna 409 `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | Teste de integração |
| FDD-AC-13 | `GET /webhooks/:id/deliveries` lista tentativas com sucesso/falha, payload, resposta e `durationMs`, e rejeita `pageSize > 100` | Teste de integração |
| FDD-AC-14 | Com a API parada, o worker segue entregando; com o worker parado, a API segue mudando status | Teste manual/E2E em ambiente local (`docker-compose.yml`) |
| FDD-AC-15 | `tests/orders.test.ts` e `tests/auth.test.ts` passam sem alteração de asserções | `npm test` |
| FDD-AC-16 | Nenhum log contém o valor de `secret` ou `previousSecret` | Teste do logger com payload contendo esses campos |

## 14. Riscos e mitigação

| ID | Risco técnico | Prob. | Impacto | Mitigação |
|---|---|---|---|---|
| FDD-RISK-01 | **Head-of-line blocking:** com worker único e sequencial, endpoints lentos (até 10 s cada) atrasam o lote e podem estourar os 10 s para outros clientes | Média | Alto | Timeout de 10 s, backoff que tira o evento da frente, lote pequeno, métrica FDD-OBS-14 como gatilho de escala (RFC-OQ-03) |
| FDD-RISK-02 | **Ordenação violada sob retry:** evento `SHIPPED` em backoff é ultrapassado por `DELIVERED` do mesmo pedido | Média | Médio | `timestamp` no payload permite ao cliente ordenar; decisão pendente na revisão (RFC-OQ-06) |
| FDD-RISK-03 | **Timeout da transação interativa** de `changeStatus` com mais consultas | Baixa | Alto | Uma leitura indexada (`customerId, active`) + um `createMany`; medir em teste de integração |
| FDD-RISK-04 | **Secret vazada por log ou resposta** | Baixa | Alto | Redaction no Pino (FDD-OBS-09), secret só em POST/rotate, FDD-AC-05 e FDD-AC-16, revisão da Sofia |
| FDD-RISK-05 | **Reenvio duplicado** depois de crash (`PROCESSING → PENDING`) | Média | Baixo | Garantia at-least-once documentada e `X-Event-Id` estável ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)) |
| FDD-RISK-06 | **Crescimento da outbox e de `webhook_deliveries`** sem arquivamento | Alta (longo prazo) | Médio | Índices `(status, nextAttemptAt)` e `(webhookId, createdAt)`; arquivamento em feature futura (RFC-OQ-04) |
| FDD-RISK-07 | **Redirect ou URL interna** usada como destino | Baixa | Médio | `https` obrigatório, `redirect: 'manual'`; tema levado à revisão de segurança |
| FDD-RISK-08 | **Dois workers iniciados por engano** quebram a premissa de ordenação | Baixa | Médio | Claim condicional (`updateMany … where status = PENDING`) evita envio duplo do mesmo lote; runbook de deploy com uma única instância |
| FDD-RISK-09 | **URL interna/SSRF:** HTTPS e redirect manual não excluem destino loopback, privado, link-local ou resolução de DNS para destino interno | Não estimada | Alto | Política de destinos e validação sujeitas à decisão de segurança (RFC-OQ-11); não considerar mitigado apenas por validar o esquema |
| FDD-RISK-10 | **Remoção destrói histórico ou entregas pendentes** caso cascata seja adotada | Não estimada | Alto | Decidir exclusão física/lógica e retenção antes de fechar contrato/migração (RFC-OQ-12); ação referencial não está aprovada |
| FDD-RISK-11 | **Acesso cruzado entre customers** se o caller puder escolher `customerId` sem checagem de vínculo | Não estimada | Alto | Resolver o modelo de autorização por customer antes de implementar CRUD/histórico (RFC-OQ-10) |

## 15. Plano de implementação

Estimativa da Tech Lead: **3 sprints, incluindo a revisão de segurança** ([09:46] Larissa, [09:47] Larissa).

A ordem segue a quebra feita na reunião: modelagem (1 sprint) → worker e retry (1 sprint) → CRUD e deliveries (½) → integração no `order.service` e testes ponta a ponta (½) → HMAC, schemas e validações ("mais um pouco") → revisão da Sofia no fim.

| ID | Etapa | Entregas | Itens |
|---|---|---|---|
| FDD-PLAN-01 | Sprint 1 | Modelagem da outbox e da DLQ: schema Prisma, migração, `publishWebhookEvent` com testes unitários | FDD-INT-16, FDD-DADOS-01..07 |
| FDD-PLAN-02 | Sprint 2 | Worker e retry: `src/worker.ts`, claim, envio, backoff, DLQ, replay | FDD-FLUXO-02..04, FDD-AC-08..10 |
| FDD-PLAN-03 | Sprint 3 (½) | CRUD de configuração, rotação de secret e histórico de entregas | FDD-CONTRATO-01..07, FDD-AC-05, FDD-AC-11..13 |
| FDD-PLAN-04 | Sprint 3 (½) | Integração em `OrderService.changeStatus` e testes ponta a ponta (atomicidade, API ↔ worker) | FDD-INT-01, FDD-FLUXO-01, FDD-AC-01..03, FDD-AC-14, FDD-AC-15 |
| FDD-PLAN-05 | Fechamento | HMAC (`webhook.signer.ts`), schemas e validações, e **revisão de segurança da Sofia (≥ 2 dias úteis)** antes do deploy | FDD-CONTRATO-08, FDD-AC-04, FDD-AC-06, FDD-AC-07, FDD-AC-16, FDD-DEP-08 |

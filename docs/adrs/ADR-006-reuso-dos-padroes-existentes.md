# ADR-006 — Reuso dos padrões existentes do projeto no módulo de webhooks

## Status

**Aceito**

- **Data da decisão:** reunião técnica de quinta-feira, 09:27–09:30 (data civil não informada na transcrição)
- **Decisores:** Larissa (Tech Lead), Bruno (Eng. Pleno, Pedidos)
- **Consultados:** Diego (Eng. Sênior, Plataforma)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)

## Contexto

O OMS já tem convenções consolidadas, verificáveis no código:

| Padrão | Onde está no código |
|---|---|
| Um módulo por domínio, com `controller`, `service`, `repository`, `routes` e `schemas` | `src/modules/orders/`, `src/modules/customers/`, `src/modules/products/` |
| Montagem das dependências por injeção manual via construtor | `buildControllers` em `src/app.ts` |
| Registro dos routers sob `/api/v1` | `buildApiRouter` em `src/routes/index.ts` |
| Hierarquia de erros com `statusCode` + `errorCode` | `AppError` em `src/shared/errors/app-error.ts`; `InsufficientStockError` e `InvalidStatusTransitionError` em `src/shared/errors/http-errors.ts` |
| Códigos de erro em UPPER_SNAKE_CASE | `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION` em `src/shared/errors/http-errors.ts` |
| Tratamento centralizado de `AppError`, `ZodError` e erros do Prisma | `errorMiddleware` em `src/middlewares/error.middleware.ts` |
| Validação de entrada com Zod | `validate()` em `src/middlewares/validate.middleware.ts` + `*.schemas.ts` |
| Autenticação JWT e autorização por role | `authenticate` e `requireRole` em `src/middlewares/auth.middleware.ts` |
| Logger estruturado Pino com redaction | `src/shared/logger/index.ts` |
| Paginação padronizada | `paginated()` em `src/shared/http/response.ts` |
| IDs UUID | `@default(uuid()) @db.Char(36)` em `prisma/schema.prisma` |

A dúvida era se a nova feature, que traz um processo novo (worker), justificaria ferramentas ou estruturas próprias.

## Decisão

**Reuso máximo do que já existe. O webhook é um módulo igual aos outros** ([09:30] Larissa):

1. **Módulo `src/modules/webhooks/`** com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts` e `webhook.schemas.ts` ([09:27] Bruno). A lógica do worker também fica no módulo (`webhook.worker.ts`), e a entry point fica em `src/worker.ts` ([09:28] Bruno).
2. **Erros estendem `AppError`, seguindo `InsufficientStockError`, com prefixo `WEBHOOK_`** em todos os códigos do módulo (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) ([09:28] Bruno, [09:29] Larissa).
3. **O `errorMiddleware` não é alterado.** Como as novas classes são `AppError`, ele já serializa `{ error: { code, message, details } }` ([09:29] Bruno).
4. **Pino como único logger**, sem biblioteca nova de logging ([09:29] Bruno).
5. **Schemas Zod** no mesmo formato dos módulos existentes, aplicados via `validate()` ([09:30] Larissa).
6. **`requireRole('ADMIN')`** existente para o endpoint de replay ([09:36] Larissa).
7. **Worker com `PrismaClient` próprio**, mesma `DATABASE_URL`, criado pelo `createPrismaClient()` de `src/config/database.ts` ([09:30] Bruno).
8. **IDs UUID** em todas as novas tabelas ([09:51] Larissa).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Estrutura própria para webhooks (ex.: pacote separado, framework de jobs)** | Diverge do padrão de módulos que o time já conhece ([09:27] Bruno). Nenhum ganho justificou isso na reunião. |
| **Adicionar biblioteca nova (logger dedicado ao worker, lib de filas/jobs)** | "Não vamos botar nada novo" ([09:29] Bruno). Pino e Prisma cobrem a necessidade, e a fila é a outbox ([ADR-001](ADR-001-outbox-no-mysql.md)). |
| **Reusar o mesmo `PrismaClient` da API no worker** | Não é possível: `PrismaClient` é por processo, e o worker é outro processo Node ([09:30] Bruno). |
| **Usar os códigos genéricos existentes (`NOT_FOUND`, `CONFLICT`) sem prefixo** | Perde a especificidade que o time quer para diagnóstico e para o cliente ([09:28] Bruno, [09:29] Larissa). Os códigos genéricos continuam valendo para autenticação e autorização (`UNAUTHORIZED`, `FORBIDDEN`). |

## Consequências

**Positivas**
- A curva de aprendizado é mínima: quem conhece `orders` navega em `webhooks`.
- O contrato de erro para quem consome a API continua o mesmo.
- Nenhuma dependência nova de runtime: HTTP via `fetch` nativo do Node ≥ 20 (`engines` em `package.json`), HMAC via `crypto` nativo, `uuid` já presente.

**Negativas**
- **A validação Zod via `validate()` sempre responde `code: VALIDATION_ERROR`** (`src/middlewares/validate.middleware.ts`). Validações de schema como `https` obrigatório não saem com código `WEBHOOK_*` de primeiro nível. O código `WEBHOOK_INVALID_URL` vai em `details[].message`. O FDD detalha essa convenção.
- `NotFoundError` tem código fixo `NOT_FOUND`. As classes `WEBHOOK_*` de 404 estendem `AppError` diretamente.
- O worker herda as limitações da stack atual: não há biblioteca de métricas nem de tracing, então a observabilidade se apoia em logs estruturados (ver FDD).

**Trade-off explícito:** trocamos a liberdade de escolher ferramentas específicas para processamento assíncrono por consistência e menor custo de manutenção. Algumas convenções (como `VALIDATION_ERROR`) limitam a granularidade de códigos que o módulo expõe.

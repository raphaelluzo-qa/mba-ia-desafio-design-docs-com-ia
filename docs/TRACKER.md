# Tracker de Rastreabilidade — Webhooks de Notificação de Pedidos

Referência cruzada entre cada item registrado no pacote ([PRD](PRD.md), [RFC](RFC.md), [FDD](FDD.md), [ADRs](adrs/)) e sua origem em [`TRANSCRICAO.md`](../TRANSCRICAO.md) ou no código-fonte.

**Convenções**
- **Fonte:** `TRANSCRICAO` ou `CODIGO`.
- **Localização:** para `TRANSCRICAO`, `[hh:mm] Nome` do falante, com a fala principal primeiro e falas de apoio depois de `;`. Para `CODIGO`, o caminho do arquivo no repositório.
- **IDs dos ADRs:** `ADR-NNN` é a decisão principal. `ADR-NNN-Dxx`, `-ALT-xx`, `-CON-xx` e `-CTX-xx` são sub-itens (decisão complementar, alternativa, consequência, contexto) do mesmo arquivo.
- Itens marcados **(derivado)** são inferências explícitas a partir da fala indicada. O documento de origem os sinaliza como interpretação ou questão em aberto, não como decisão da reunião.

## ADRs

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão outbox: inserir evento em `webhook_outbox` na mesma transação que atualiza orders e history | TRANSCRICAO | [09:06] Diego; [09:08] Larissa |
| ADR-001-CTX-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | `changeStatus` roda em `$transaction` com validação, estoque, update de orders e insert de history | CODIGO | src/modules/orders/order.service.ts |
| ADR-001-CTX-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | Clientes B2B pediram notificação de mudança de status em vez de polling | TRANSCRICAO | [09:00] Marcos |
| ADR-001-D01 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o tx, sem injetar repository | TRANSCRICAO | [09:41] Bruno; [09:41] Diego |
| ADR-001-D02 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Falha no insert da outbox faz rollback; não pode haver status mudado sem evento | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| ADR-001-D03 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Índices da outbox por status e `created_at` | TRANSCRICAO | [09:08] Diego |
| ADR-001-D04 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | IDs UUID nas novas tabelas | TRANSCRICAO | [09:51] Larissa |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Disparo HTTP síncrono dentro de `changeStatus` | TRANSCRICAO | [09:04] Bruno; [09:06] Diego |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Redis Streams / fila externa (infra nova, overengineering) | TRANSCRICAO | [09:07] Larissa; [09:07] Diego |
| ADR-001-ALT-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Injetar repository de webhooks inteiro no OrderService | TRANSCRICAO | [09:41] Diego |
| ADR-001-CON-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Atomicidade forte em troca de acoplar mudança de status à saúde da outbox | TRANSCRICAO | [09:41] Diego |
| ADR-001-CON-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Fora de Escopo | Arquivamento de linhas entregues após ~30 dias fora desta feature | TRANSCRICAO | [09:08] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em processo separado lendo a outbox por polling de 2 s | TRANSCRICAO | [09:09] Diego; [09:10] Larissa |
| ADR-002-CTX-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Contexto | Banco é MySQL (sem LISTEN/NOTIFY) | CODIGO | prisma/schema.prisma |
| ADR-002-CTX-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Contexto | API sobe por `src/server.ts`; restart da API derrubaria worker embutido | CODIGO | src/server.ts |
| ADR-002-D01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Entry point `src/worker.ts` e script `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| ADR-002-D02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Lógica de processamento em `src/modules/webhooks/webhook.worker.ts` | TRANSCRICAO | [09:28] Bruno |
| ADR-002-D03 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Mesmo banco e stack, `PrismaClient` próprio por processo | TRANSCRICAO | [09:30] Bruno; [09:11] Bruno; [09:11] Diego |
| ADR-002-D04 | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Single-worker; ordenação implícita por `order_id` via `created_at` | TRANSCRICAO | [09:12] Diego |
| ADR-002-D05 | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Limitação conhecida: sem garantia de ordering global | TRANSCRICAO | [09:13] Larissa; [09:14] Marcos |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Trigger no banco para acordar o worker | TRANSCRICAO | [09:09] Bruno; [09:09] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Worker no mesmo processo da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-03 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Múltiplos workers (particionar por order_id / lock pessimista) | TRANSCRICAO | [09:13] Diego |
| ADR-002-CON-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Latência mínima de ~2 s aceita | TRANSCRICAO | [09:10] Larissa; [09:10] Marcos |
| ADR-002-CON-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Head-of-line blocking do worker único sequencial (derivado) | TRANSCRICAO | [09:12] Diego |
| ADR-002-CON-03 | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Ordenação por pedido pode ser violada durante retry (derivado) | TRANSCRICAO | [09:13] Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Retry com backoff exponencial 1m/5m/30m/2h/12h, teto de 5, depois DLQ | TRANSCRICAO | [09:17] Larissa; [09:15] Diego |
| ADR-003-D01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Cinco intervalos listados; soma ~14,6 h se todos usados, sem definir total de chamadas | TRANSCRICAO | [09:17] Diego; [09:17] Larissa |
| ADR-003-D02 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Questão em Aberto | “5 tentativas” sem esclarecer se inclui envio inicial | TRANSCRICAO | [09:15] Diego; [09:16] Larissa |
| ADR-003-D03 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Questão em Aberto | Contagem total não fechada: a reunião cita 5 tentativas e 5 intervalos | TRANSCRICAO | [09:16] Diego; [09:17] Diego; [09:17] Larissa |
| ADR-003-D04 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Timeout de 10 s conta como falha com retry | TRANSCRICAO | [09:42] Diego |
| ADR-003-D05 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | DLQ em tabela separada `webhook_dead_letter` com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| ADR-003-D06 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Replay manual via `POST /admin/webhooks/dead-letter/:id/replay`, recolocando como pendente | TRANSCRICAO | [09:18] Diego; [09:19] Larissa |
| ADR-003-D07 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Restrição | Replay exige role ADMIN | TRANSCRICAO | [09:36] Sofia; [09:36] Larissa |
| ADR-003-D08 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Reaproveitar `requireRole` existente para o replay | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-003-D09 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Restrição | Replay registra em log quem executou (auditoria) | TRANSCRICAO | [09:36] Sofia |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa Descartada | Retry indefinido com backoff | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa Descartada | 3 tentativas (cobre só ~30 min) | TRANSCRICAO | [09:16] Bruno; [09:16] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa Descartada | Marcar `failed` na própria outbox em vez de tabela de DLQ | TRANSCRICAO | [09:17] Larissa; [09:18] Diego |
| ADR-003-ALT-04 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa Descartada | Replay por qualquer usuário autenticado | TRANSCRICAO | [09:36] Sofia |
| ADR-003-CON-01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Contexto | PM considera indisponibilidade de 15 h um problema sério do cliente; não define contagem de chamadas | TRANSCRICAO | [09:17] Marcos |
| ADR-003-CON-02 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Trade-off | Sem aviso automático ao cliente; e-mail em fase futura | TRANSCRICAO | [09:37] Larissa |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com grace de 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-CTX-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Contexto | Webhooks apenas outbound | TRANSCRICAO | [09:03] Sofia; [09:02] Marcos |
| ADR-004-CTX-02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Contexto | Cliente precisa validar origem e integridade | TRANSCRICAO | [09:19] Sofia |
| ADR-004-CTX-03 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Contexto | Histórico de cliente que vazou secret em log | TRANSCRICAO | [09:22] Diego |
| ADR-004-D01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Assinatura enviada no header `X-Signature` | TRANSCRICAO | [09:20] Sofia |
| ADR-004-D02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret única por endpoint, nunca global | TRANSCRICAO | [09:21] Sofia |
| ADR-004-D03 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| ADR-004-D04 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Configuração armazena url, secret, customer_id e estado ativo | TRANSCRICAO | [09:21] Bruno; [09:21] Sofia |
| ADR-004-D05 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Rotação via API; antiga válida em paralelo por 24 h | TRANSCRICAO | [09:21] Sofia |
| ADR-004-D06 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | URL obrigatoriamente `https`, validada no schema Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-004-D07 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | Revisão de segurança de pelo menos 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia; [09:47] Larissa |
| ADR-004-ALT-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa Descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT-02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa Descartada | Secret sem rotação | TRANSCRICAO | [09:22] Diego |
| ADR-004-ALT-03 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa Descartada | Aceitar URLs http | TRANSCRICAO | [09:23] Sofia |
| ADR-004-ALT-04 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa Descartada | Outros algoritmos de HMAC | TRANSCRICAO | [09:20] Bruno; [09:20] Sofia |
| ADR-004-CON-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Secret precisa ser recuperável (não hash como `passwordHash`) | CODIGO | prisma/schema.prisma |
| ADR-004-CON-02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | Secret incluída no `redact` do Pino | CODIGO | src/shared/logger/index.ts |
| ADR-004-CON-03 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Detecção de replay attack pelo cliente via `X-Timestamp` é opcional | TRANSCRICAO | [09:44] Diego |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | Garantia at-least-once com `X-Event-Id` para dedup pelo cliente | TRANSCRICAO | [09:26] Larissa; [09:24] Diego |
| ADR-005-D01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | `event_id` UUID gerado na entrada da outbox, único por evento | TRANSCRICAO | [09:25] Diego |
| ADR-005-D02 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | `event_id` estável em retries e replays (derivado) | TRANSCRICAO | [09:18] Diego |
| ADR-005-D03 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | `event_id` também no corpo do payload | TRANSCRICAO | [09:43] Diego |
| ADR-005-D04 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | Header `X-Webhook-Id` identifica o cadastro | TRANSCRICAO | [09:44] Sofia; [09:45] Diego |
| ADR-005-D05 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Dependência | PM documenta dedup com destaque no portal | TRANSCRICAO | [09:26] Marcos |
| ADR-005-ALT-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Alternativa Descartada | Exactly-once (coordenação dos dois lados) | TRANSCRICAO | [09:25] Diego |
| ADR-005-ALT-02 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Alternativa Descartada | At-most-once sem retry (plausível, incompatível com retry) | TRANSCRICAO | [09:14] Larissa |
| ADR-005-ALT-03 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Alternativa Descartada | Não enviar identificador de evento | TRANSCRICAO | [09:25] Bruno |
| ADR-005-CON-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de deduplicação transferida ao cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-005-CON-02 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Padrão de mercado (Stripe, GitHub), resolve 99% dos casos | TRANSCRICAO | [09:25] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, Zod, códigos de erro | TRANSCRICAO | [09:30] Larissa |
| ADR-006-CTX-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Contexto | Montagem de dependências por construtor em `buildControllers` | CODIGO | src/app.ts |
| ADR-006-CTX-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Contexto | Routers montados sob `/api/v1` em `buildApiRouter` | CODIGO | src/routes/index.ts |
| ADR-006-CTX-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Contexto | Classe base `AppError` com statusCode e errorCode | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-CTX-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Contexto | `InsufficientStockError`/`InvalidStatusTransitionError` com códigos UPPER_SNAKE | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-CTX-05 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Contexto | Paginação padronizada `paginated()` | CODIGO | src/shared/http/response.ts |
| ADR-006-D01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Módulo `src/modules/webhooks` com controller, service, repository, routes, schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-D02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Erros estendem AppError com códigos `WEBHOOK_*` | TRANSCRICAO | [09:28] Bruno; [09:29] Larissa |
| ADR-006-D03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Error middleware centralizado sem alteração | TRANSCRICAO | [09:29] Bruno |
| ADR-006-D04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Pino como único logger | TRANSCRICAO | [09:29] Bruno |
| ADR-006-D05 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Schemas Zod via `validate()` | TRANSCRICAO | [09:30] Larissa |
| ADR-006-D06 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso do `requireRole` para ADMIN | TRANSCRICAO | [09:36] Larissa |
| ADR-006-D07 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Worker cria PrismaClient via `createPrismaClient()` | CODIGO | src/config/database.ts |
| ADR-006-D08 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | UUID em todas as novas tabelas | TRANSCRICAO | [09:51] Larissa |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa Descartada | Estrutura própria / framework de jobs fora do padrão de módulos | TRANSCRICAO | [09:27] Bruno |
| ADR-006-ALT-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa Descartada | Adicionar biblioteca nova | TRANSCRICAO | [09:29] Bruno |
| ADR-006-ALT-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa Descartada | Reusar o mesmo PrismaClient da API no worker | TRANSCRICAO | [09:30] Bruno; [09:29] Diego |
| ADR-006-ALT-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa Descartada | Usar códigos genéricos sem prefixo `WEBHOOK_` | TRANSCRICAO | [09:29] Larissa |
| ADR-006-CON-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Trade-off | `validate()` sempre responde `VALIDATION_ERROR`; código WEBHOOK_ vai em details | CODIGO | src/middlewares/validate.middleware.ts |
| ADR-006-CON-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Trade-off | `NotFoundError` tem código fixo `NOT_FOUND`; classes 404 estendem AppError | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-CON-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Trade-off | Sem dependência nova: fetch nativo do Node ≥ 20, crypto nativo, uuid existente | CODIGO | package.json |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Decisão | Payload enxuto como snapshot na inserção; filtro de eventos na inserção | TRANSCRICAO | [09:52] Bruno; [09:34] Bruno |
| ADR-007-CTX-01 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Contexto | `OrderService.delete` permite apagar pedidos PENDING/CANCELLED | CODIGO | src/modules/orders/order.service.ts |
| ADR-007-D01 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Decisão | Filtro por status aplicado na inserção; nenhum interessado ⇒ nenhuma linha | TRANSCRICAO | [09:34] Bruno; [09:34] Diego |
| ADR-007-D02 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Decisão | Payload renderizado (snapshot) na inserção | TRANSCRICAO | [09:52] Larissa; [09:52] Diego |
| ADR-007-D03 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Decisão | Campos: event_id, event_type, timestamp, order_id, order_number, from/to_status, customer_id, total_cents | TRANSCRICAO | [09:43] Diego |
| ADR-007-D04 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Restrição | Sem `items`; detalhes via GET /orders/:id | TRANSCRICAO | [09:43] Diego; [09:44] Bruno |
| ADR-007-ALT-01 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Alternativa Descartada | Filtrar na hora do envio | TRANSCRICAO | [09:34] Diego; [09:34] Bruno |
| ADR-007-ALT-02 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Alternativa Descartada | Guardar só order_id e renderizar no envio | TRANSCRICAO | [09:51] Bruno; [09:52] Larissa |
| ADR-007-ALT-03 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Alternativa Descartada | Payload completo com items | TRANSCRICAO | [09:43] Diego |
| ADR-007-CON-01 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Trade-off | Payload pequeno mantém limite de 64 KB distante | TRANSCRICAO | [09:24] Diego |
| ADR-007-CON-02 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Trade-off | Mudança de filtro não é retroativa nem afeta eventos já enfileirados (derivado) | TRANSCRICAO | [09:34] Bruno |

## RFC

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| RFC-META-01 | docs/RFC.md | Metadado | Revisores = participantes da reunião; status em revisão antes de codar | TRANSCRICAO | [09:50] Larissa |
| RFC-CTX-01 | docs/RFC.md | Contexto | Clientes fazem polling em GET /orders; integração lenta e cara; risco de churn da Atlas | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | docs/RFC.md | Requisito Não Funcional | "Tempo real" = abaixo de 10 s | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-03 | docs/RFC.md | Contexto | Ausência de mecanismo de notificação; única rota é PATCH /orders/:id/status | CODIGO | src/modules/orders/order.routes.ts |
| RFC-CTX-04 | docs/RFC.md | Contexto | Transação de mudança de status já é pesada | TRANSCRICAO | [09:04] Bruno |
| RFC-PROP-01 | docs/RFC.md | Decisão | Componente outbox `webhook_outbox` | TRANSCRICAO | [09:06] Diego |
| RFC-PROP-02 | docs/RFC.md | Decisão | Componente publisher `publishWebhookEvent` | TRANSCRICAO | [09:41] Bruno |
| RFC-PROP-03 | docs/RFC.md | Decisão | Componente worker `src/worker.ts`, polling 2 s, timeout 10 s | TRANSCRICAO | [09:11] Larissa; [09:09] Diego; [09:42] Diego |
| RFC-PROP-04 | docs/RFC.md | Decisão | Componente retry + DLQ com replay ADMIN | TRANSCRICAO | [09:48] Larissa |
| RFC-PROP-05 | docs/RFC.md | Decisão | Componente segurança: HMAC, secret por endpoint, rotação, https | TRANSCRICAO | [09:22] Sofia; [09:23] Sofia |
| RFC-PROP-06 | docs/RFC.md | Decisão | Semântica at-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| RFC-PROP-07 | docs/RFC.md | Decisão | API de configuração (CRUD, rotação, deliveries, replay) | TRANSCRICAO | [09:31] Marcos; [09:33] Bruno; [09:34] Marcos |
| RFC-PRINC-01 | docs/RFC.md | Trade-off | Consistência antes de latência (2 s aceitos) | TRANSCRICAO | [09:10] Larissa |
| RFC-PRINC-02 | docs/RFC.md | Restrição | Zero infraestrutura nova | TRANSCRICAO | [09:07] Diego; [09:29] Bruno |
| RFC-PRINC-03 | docs/RFC.md | Decisão | Isolamento por endpoint (secret, filtro, identificação) | TRANSCRICAO | [09:21] Sofia; [09:33] Bruno; [09:44] Sofia |
| RFC-PRINC-04 | docs/RFC.md | Decisão | Webhook é um módulo como os outros | TRANSCRICAO | [09:30] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Disparo HTTP síncrono em changeStatus | TRANSCRICAO | [09:04] Bruno; [09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams / fila externa | TRANSCRICAO | [09:07] Larissa; [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Trigger do banco para notificar o worker | TRANSCRICAO | [09:09] Bruno; [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Worker no mesmo processo da API | TRANSCRICAO | [09:11] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa Descartada | Retry indefinido ou só 3 tentativas | TRANSCRICAO | [09:15] Diego; [09:16] Bruno; [09:16] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa Descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-07 | docs/RFC.md | Alternativa Descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| RFC-OQ-01 | docs/RFC.md | Questão em Aberto | Rate limiting de saída por cliente: observar e decidir depois | TRANSCRICAO | [09:38] Diego; [09:39] Larissa |
| RFC-OQ-02 | docs/RFC.md | Questão em Aberto | Aviso por e-mail em falhas repetidas: próxima fase | TRANSCRICAO | [09:37] Marcos; [09:37] Larissa |
| RFC-OQ-03 | docs/RFC.md | Questão em Aberto | Escala para múltiplos workers (partição por order_id ou lock) | TRANSCRICAO | [09:13] Bruno; [09:13] Diego |
| RFC-OQ-04 | docs/RFC.md | Questão em Aberto | Arquivamento das linhas entregues (~30 dias) | TRANSCRICAO | [09:08] Diego |
| RFC-OQ-05 | docs/RFC.md | Questão em Aberto | Endurecer autorização do CRUD de webhooks | TRANSCRICAO | [09:37] Sofia |
| RFC-OQ-06 | docs/RFC.md | Questão em Aberto | Ordenação por pedido durante retry (derivado) | TRANSCRICAO | [09:12] Diego; [09:50] Larissa |
| RFC-OQ-07 | docs/RFC.md | Questão em Aberto | Detalhes para revisão de segurança: formato X-Signature no grace, secret em repouso, X-Timestamp assinado (derivado) | TRANSCRICAO | [09:46] Sofia |
| RFC-OQ-08 | docs/RFC.md | Questão em Aberto | Confirmar se cinco tentativas incluem envio inicial e quando mover à DLQ | TRANSCRICAO | [09:16] Diego; [09:17] Diego; [09:17] Larissa |
| RFC-OQ-09 | docs/RFC.md | Questão em Aberto | Definir perfil de carga e critério para validar alvo de entrega < 10 s | TRANSCRICAO | [09:02] Marcos; [09:09] Diego; [09:12] Diego |
| RFC-OQ-10 | docs/RFC.md | Questão em Aberto | Definir autorização/ownership do usuário sobre customer; autenticação não determina isolamento | TRANSCRICAO | [09:32] Larissa; [09:32] Bruno; [09:37] Sofia |
| RFC-OQ-11 | docs/RFC.md | Questão de Segurança (derivada) | HTTPS não define política para destinos internos ou privados; requer decisão de segurança | TRANSCRICAO | [09:23] Sofia |
| RFC-OQ-12 | docs/RFC.md | Questão em Aberto | Definir hard/soft delete, retenção e efeito em outbox, deliveries e DLQ | TRANSCRICAO | [09:08] Diego; [09:33] Bruno |
| RFC-IMP-01 | docs/RFC.md | Impacto | changeStatus ganha leitura + N inserts; falha faz rollback | TRANSCRICAO | [09:40] Bruno |
| RFC-IMP-02 | docs/RFC.md | Impacto | Novas tabelas e polling de 2 s no banco | TRANSCRICAO | [09:18] Diego; [09:09] Diego |
| RFC-IMP-03 | docs/RFC.md | Impacto | Novo processo `npm run worker` para operar | TRANSCRICAO | [09:11] Larissa |
| RFC-IMP-04 | docs/RFC.md | Impacto | Clientes precisam validar HMAC, deduplicar e usar https; documentação no portal | TRANSCRICAO | [09:26] Marcos; [09:40] Marcos |
| RFC-IMP-05 | docs/RFC.md | Impacto | Secrets armazenadas pela plataforma; revisão da Sofia obrigatória | TRANSCRICAO | [09:46] Sofia |
| RFC-IMP-06 | docs/RFC.md | Impacto | Prazo de 3 sprints; Atlas espera fim de novembro | TRANSCRICAO | [09:45] Marcos; [09:47] Larissa |
| RFC-RISK-01 | docs/RFC.md | Risco | Head-of-line blocking no worker único | TRANSCRICAO | [09:42] Diego; [09:12] Diego |
| RFC-RISK-02 | docs/RFC.md | Risco | Cliente não deduplica | TRANSCRICAO | [09:25] Sofia |
| RFC-RISK-03 | docs/RFC.md | Risco | Vazamento de secret do lado do cliente | TRANSCRICAO | [09:22] Diego |
| RFC-RISK-04 | docs/RFC.md | Risco | Outbox bloquear mudanças de status | TRANSCRICAO | [09:41] Diego |
| RFC-RISK-05 | docs/RFC.md | Risco | Prazo e risco de churn da Atlas | TRANSCRICAO | [09:00] Marcos |
| RFC-RISK-06 | docs/RFC.md | Risco | Acesso cruzado entre customers sem ownership documentado | TRANSCRICAO | [09:32] Larissa; [09:32] Bruno |
| RFC-RISK-07 | docs/RFC.md | Risco de Segurança (derivado) | HTTPS/redirect manual não definem restrições contra SSRF/destinos internos | TRANSCRICAO | [09:23] Sofia |
| RFC-RISK-08 | docs/RFC.md | Risco (derivado) | Política de remoção pode apagar histórico ou entregas pendentes | TRANSCRICAO | [09:08] Diego; [09:33] Bruno |
| RFC-NEXT-01 | docs/RFC.md | Próximo Passo | Sessão de revisão com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |

## FDD

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-CTX-01 | docs/FDD.md | Contexto | Máquina de estados de pedido (transições válidas) | CODIGO | src/modules/orders/order.status.ts |
| FDD-OBJ-01 | docs/FDD.md | Objetivo Técnico | Atomicidade entre status e evento | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-02 | docs/FDD.md | Objetivo Técnico | Alvo < 10 s; perfil de carga e critério estatístico pendentes | TRANSCRICAO | [09:02] Marcos; [09:09] Diego |
| FDD-OBJ-03 | docs/FDD.md | Objetivo Técnico | Nenhum evento descartado silenciosamente (entregue ou DLQ) | TRANSCRICAO | [09:15] Diego |
| FDD-OBJ-04 | docs/FDD.md | Objetivo Técnico | 100% das requisições com X-Signature HMAC-SHA256 | TRANSCRICAO | [09:20] Sofia |
| FDD-OBJ-05 | docs/FDD.md | Objetivo Técnico | Zero dependência nova e zero mudança em rotas existentes | TRANSCRICAO | [09:29] Bruno |
| FDD-ESC-01 | docs/FDD.md | Escopo | Módulo src/modules/webhooks com CRUD, rotação, deliveries, replay | TRANSCRICAO | [09:27] Bruno |
| FDD-ESC-02 | docs/FDD.md | Escopo | Quatro tabelas novas (config, outbox, deliveries, DLQ) | TRANSCRICAO | [09:18] Diego; [09:21] Bruno; [09:34] Marcos |
| FDD-ESC-03 | docs/FDD.md | Escopo | publishWebhookEvent chamada por changeStatus | TRANSCRICAO | [09:41] Bruno |
| FDD-ESC-04 | docs/FDD.md | Escopo | src/worker.ts + npm run worker | TRANSCRICAO | [09:11] Larissa |
| FDD-ESC-05 | docs/FDD.md | Escopo | Evento único `order.status_changed` | TRANSCRICAO | [09:43] Diego |
| FDD-EXC-01 | docs/FDD.md | Fora de Escopo | Webhooks inbound | TRANSCRICAO | [09:02] Marcos |
| FDD-EXC-02 | docs/FDD.md | Fora de Escopo | E-mail em falha de webhook | TRANSCRICAO | [09:37] Larissa |
| FDD-EXC-03 | docs/FDD.md | Fora de Escopo | Rate limiting de saída | TRANSCRICAO | [09:39] Larissa |
| FDD-EXC-04 | docs/FDD.md | Fora de Escopo | Dashboard visual | TRANSCRICAO | [09:40] Larissa |
| FDD-EXC-05 | docs/FDD.md | Fora de Escopo | Múltiplos workers | TRANSCRICAO | [09:13] Diego |
| FDD-EXC-06 | docs/FDD.md | Fora de Escopo | Arquivamento da outbox | TRANSCRICAO | [09:08] Diego |
| FDD-EXC-07 | docs/FDD.md | Fora de Escopo | Evento na criação do pedido (PENDING inicial não passa por changeStatus) | CODIGO | src/modules/orders/order.service.ts |
| FDD-EXC-08 | docs/FDD.md | Fora de Escopo | Ordenação global e exactly-once | TRANSCRICAO | [09:13] Larissa; [09:25] Diego |
| FDD-CONST-01 | docs/FDD.md | Configuração | BACKOFF_SCHEDULE_MS 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| FDD-CONST-02 | docs/FDD.md | Configuração Pendente | Não fixar limite ou total de chamadas até resolver RFC-OQ-08 | TRANSCRICAO | [09:16] Diego; [09:17] Diego; [09:17] Larissa |
| FDD-CONST-03 | docs/FDD.md | Configuração | MAX_PAYLOAD_BYTES = 64 KB | TRANSCRICAO | [09:24] Larissa |
| FDD-CONST-04 | docs/FDD.md | Configuração | SECRET_GRACE_PERIOD_MS = 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONST-05 | docs/FDD.md | Configuração | EVENT_TYPE = order.status_changed | TRANSCRICAO | [09:43] Diego |
| FDD-CONST-06 | docs/FDD.md | Restrição | Status inscrevíveis = destinos de alguma transição (exclui PENDING) | CODIGO | src/modules/orders/order.status.ts |
| FDD-ENV-01 | docs/FDD.md | Configuração | WEBHOOK_POLL_INTERVAL_MS = 2000 | TRANSCRICAO | [09:09] Diego |
| FDD-ENV-02 | docs/FDD.md | Configuração | WEBHOOK_HTTP_TIMEOUT_MS = 10000 | TRANSCRICAO | [09:42] Diego |
| FDD-ENV-03 | docs/FDD.md | Configuração | WEBHOOK_BATCH_SIZE = 10 ("batch pequeno") | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-01 | docs/FDD.md | Modelo de Dados | Tabela `webhooks` com url, secret, customer_id, active | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-02 | docs/FDD.md | Modelo de Dados | Uma linha de outbox por evento × webhook, eventId compartilhado | TRANSCRICAO | [09:25] Diego |
| FDD-DADOS-03 | docs/FDD.md | Modelo de Dados | Índices da outbox por status e created_at | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-04 | docs/FDD.md | Modelo de Dados | Payload serializado gravado como snapshot | TRANSCRICAO | [09:52] Larissa |
| FDD-DADOS-05 | docs/FDD.md | Modelo de Dados | DLQ separada com payload, motivo, timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-06 | docs/FDD.md | Modelo de Dados | `webhook_deliveries` registra cada tentativa | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-07 | docs/FDD.md | Modelo de Dados | url e secret lidas no envio; rotação vale para pendentes | TRANSCRICAO | [09:21] Sofia |
| FDD-DADOS-08 | docs/FDD.md | Modelo de Dados | PKs UUID Char(36) e DATETIME(3) como nas tabelas existentes | CODIGO | prisma/migrations/20260519182739_init/migration.sql |
| FDD-CONTRATO-00 | docs/FDD.md | Restrição | customerId no body (criação) ou path (listagem), não derivado do JWT | TRANSCRICAO | [09:32] Larissa; [09:32] Bruno |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks — cria e devolve secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/customers/:customerId/webhooks — lista sem secret | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id — edita url/events/active | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id — remove | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret — grace de 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries — histórico paginado | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay — ADMIN | TRANSCRICAO | [09:35] Diego; [09:36] Sofia |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Requisição outbound: payload + X-Event-Id, X-Webhook-Id, X-Timestamp, X-Signature, Content-Type | TRANSCRICAO | [09:44] Diego; [09:44] Sofia; [09:43] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Autorização: CRUD qualquer role autenticada; replay só ADMIN | TRANSCRICAO | [09:37] Sofia; [09:36] Sofia |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Formato de erro `{error:{code,message,details}}` e paginação `{data,pagination}` | CODIGO | src/middlewares/error.middleware.ts |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Criação do evento na outbox dentro de changeStatus | TRANSCRICAO | [09:40] Bruno; [09:41] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Processamento pelo worker (recover, claim, envio, registro) | TRANSCRICAO | [09:09] Diego; [09:08] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Retry com backoff (tabela de tentativas) | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Entrada na DLQ e replay com auditoria | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Criação de webhook e rotação de secret | TRANSCRICAO | [09:31] Marcos; [09:21] Sofia |
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND (404) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_CUSTOMER_NOT_FOUND (404) | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND (404) | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED (409) | TRANSCRICAO | [09:36] Sofia |
| FDD-ERR-05 | docs/FDD.md | Erro | WEBHOOK_INACTIVE (422) no replay | TRANSCRICAO | [09:21] Bruno |
| FDD-ERR-06 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL (400, via validação Zod) | TRANSCRICAO | [09:23] Sofia; [09:28] Bruno |
| FDD-ERR-07 | docs/FDD.md | Erro | WEBHOOK_INVALID_EVENTS (400, via validação Zod) | TRANSCRICAO | [09:33] Marcos |
| FDD-ERR-08 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT (retry) | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-09 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_NETWORK_ERROR (retry) | TRANSCRICAO | [09:14] Larissa |
| FDD-ERR-10 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_HTTP_ERROR (retry) | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-11 | docs/FDD.md | Erro | WEBHOOK_MAX_ATTEMPTS_EXCEEDED (DLQ) | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-12 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (DLQ, sem HTTP) | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-13 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED (DLQ, sem HTTP) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-14 | docs/FDD.md | Erro | WEBHOOK_INACTIVE no worker (DLQ, sem HTTP) | TRANSCRICAO | [09:21] Bruno |
| FDD-ERR-15 | docs/FDD.md | Erro | Reuso de UNAUTHORIZED/FORBIDDEN/INTERNAL_SERVER_ERROR existentes | CODIGO | src/shared/errors/http-errors.ts |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout de 10 s por chamada | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Backoff fixo via nextAttemptAt | TRANSCRICAO | [09:17] Larissa |
| FDD-RES-03 | docs/FDD.md | Resiliência | DLQ/replay definidos; limite de chamadas pendente em RFC-OQ-08 | TRANSCRICAO | [09:15] Diego; [09:18] Diego |
| FDD-RES-04 | docs/FDD.md | Resiliência | Falha rápida sem retry (payload > 64 KB, inativo, sem secret) | TRANSCRICAO | [09:23] Sofia |
| FDD-RES-05 | docs/FDD.md | Resiliência | Atomicidade como fallback de consistência | TRANSCRICAO | [09:41] Diego |
| FDD-RES-06 | docs/FDD.md | Resiliência | Recuperação PROCESSING → PENDING no boot (at-least-once) | TRANSCRICAO | [09:24] Diego |
| FDD-RES-07 | docs/FDD.md | Resiliência | Isolamento de processo API × worker | TRANSCRICAO | [09:11] Diego |
| FDD-RES-08 | docs/FDD.md | Resiliência | Ticks sem sobreposição (preserva single-worker) | TRANSCRICAO | [09:12] Diego |
| FDD-RES-09 | docs/FDD.md | Resiliência | Redirect manual; 3xx é falha (https obrigatório) | TRANSCRICAO | [09:23] Sofia |
| FDD-RES-10 | docs/FDD.md | Resiliência | Erro no tick é logado e o loop continua | TRANSCRICAO | [09:11] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Log webhook_event_staged (debug, pré-commit) | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Logs webhook_worker_started/stopped | TRANSCRICAO | [09:11] Diego |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Log webhook_delivery_succeeded com durationMs | TRANSCRICAO | [09:34] Marcos |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Log webhook_delivery_retry_scheduled | TRANSCRICAO | [09:15] Diego |
| FDD-OBS-05 | docs/FDD.md | Observabilidade | Log webhook_moved_to_dead_letter | TRANSCRICAO | [09:18] Diego |
| FDD-OBS-06 | docs/FDD.md | Observabilidade | Log de auditoria webhook_dead_letter_replayed com replayedById | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-07 | docs/FDD.md | Observabilidade | Log webhook_secret_rotated sem valores | TRANSCRICAO | [09:21] Sofia |
| FDD-OBS-08 | docs/FDD.md | Observabilidade | Log webhook_worker_tick_failed | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-09 | docs/FDD.md | Observabilidade | Redaction de secret/previousSecret no Pino | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-10 | docs/FDD.md | Observabilidade | Medir latência contra alvo < 10 s após definir perfil/corte estatístico | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-11 | docs/FDD.md | Observabilidade | Métrica webhook_delivery_attempts_total (base p/ rate limiting) | TRANSCRICAO | [09:39] Diego |
| FDD-OBS-12 | docs/FDD.md | Observabilidade | Métrica webhook_delivery_duration_ms | TRANSCRICAO | [09:42] Diego |
| FDD-OBS-13 | docs/FDD.md | Observabilidade | Gauge webhook_outbox_pending_total | TRANSCRICAO | [09:07] Bruno |
| FDD-OBS-14 | docs/FDD.md | Observabilidade | Gauge idade do pendente mais antigo (gatilho de escala) | TRANSCRICAO | [09:13] Diego |
| FDD-OBS-15 | docs/FDD.md | Observabilidade | Gauge DLQ aberta (dado para decisão sobre e-mail) | TRANSCRICAO | [09:37] Larissa |
| FDD-OBS-16 | docs/FDD.md | Observabilidade | Correlação por X-Request-Id existente | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-17 | docs/FDD.md | Observabilidade | Correlação pedido → evento (orderId + eventId) | TRANSCRICAO | [09:25] Diego |
| FDD-OBS-18 | docs/FDD.md | Observabilidade | Correlação evento → tentativas (eventId + webhookId) | TRANSCRICAO | [09:25] Diego; [09:44] Sofia |
| FDD-OBS-19 | docs/FDD.md | Observabilidade | Correlação com o cliente via X-Event-Id/X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-OBS-20 | docs/FDD.md | Restrição | Sem lib de métricas/tracing na stack; métricas via logs e SQL | CODIGO | package.json |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus chama publishWebhookEvent após insert em order_status_history | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | canTransition importado para derivar status inscrevíveis | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Novas classes de erro WEBHOOK_* estendendo AppError/ConflictError/UnprocessableEntityError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Exportação das novas classes no barrel | CODIGO | src/shared/errors/index.ts |
| FDD-INT-05 | docs/FDD.md | Integração | AppError como base, sem alteração | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-06 | docs/FDD.md | Integração | errorMiddleware sem alteração; P2003 não mapeado ⇒ validar customer antes | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | authenticate + requireRole('ADMIN'); req.user.id para auditoria | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-08 | docs/FDD.md | Integração | validate() com VALIDATION_ERROR e códigos WEBHOOK_ em details | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-09 | docs/FDD.md | Integração | redactPaths + logger.child para o worker | CODIGO | src/shared/logger/index.ts |
| FDD-INT-10 | docs/FDD.md | Integração | paginated() em listagem e deliveries | CODIGO | src/shared/http/response.ts |
| FDD-INT-11 | docs/FDD.md | Integração | createPrismaClient() no worker | CODIGO | src/config/database.ts |
| FDD-INT-12 | docs/FDD.md | Integração | Novas variáveis WEBHOOK_* com default no envSchema | CODIGO | src/config/env.ts |
| FDD-INT-13 | docs/FDD.md | Integração | src/worker.ts segue bootstrap/shutdown de server.ts | CODIGO | src/server.ts |
| FDD-INT-14 | docs/FDD.md | Integração | buildControllers instancia repository/service/controller de webhooks | CODIGO | src/app.ts |
| FDD-INT-15 | docs/FDD.md | Integração | buildApiRouter monta /customers/:customerId/webhooks, /webhooks e /admin/webhooks | CODIGO | src/routes/index.ts |
| FDD-INT-16 | docs/FDD.md | Integração | Enum + 4 modelos + relação inversa em Customer; migração aditiva | CODIGO | prisma/schema.prisma |
| FDD-INT-17 | docs/FDD.md | Integração | Scripts worker e worker:start; build já inclui src/**/*.ts | CODIGO | package.json |
| FDD-INT-18 | docs/FDD.md | Integração | Limpeza das novas tabelas no beforeEach | CODIGO | tests/setup.ts |
| FDD-INT-19 | docs/FDD.md | Integração | Factory createTestWebhook; bootstrapAuthenticatedUser('ADMIN') | CODIGO | tests/helpers/factories.ts |
| FDD-INT-20 | docs/FDD.md | Integração | uuidv4 do pacote uuid para eventId | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-21 | docs/FDD.md | Integração Pendente | Middleware autentica e autoriza role, mas não define vínculo entre usuário e customer | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node ≥ 20 (fetch, AbortSignal.timeout, crypto) | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência | Só bibliotecas existentes, nenhuma nova | TRANSCRICAO | [09:29] Bruno |
| FDD-DEP-03 | docs/FDD.md | Dependência | MySQL 8.0 e mesma DATABASE_URL para API e worker | TRANSCRICAO | [09:11] Diego; [09:30] Bruno |
| FDD-DEP-04 | docs/FDD.md | Restrição | Nenhuma rota existente muda; PATCH /orders/:id/status mantém contrato | CODIGO | src/modules/orders/order.routes.ts |
| FDD-DEP-05 | docs/FDD.md | Dependência | Migração só aditiva | CODIGO | prisma/schema.prisma |
| FDD-DEP-06 | docs/FDD.md | Dependência | Ordem de deploy: migração, API, worker | TRANSCRICAO | [09:11] Diego |
| FDD-DEP-07 | docs/FDD.md | Dependência | Endpoint https do cliente, HMAC e dedup; documentação no portal | TRANSCRICAO | [09:26] Marcos; [09:40] Marcos |
| FDD-DEP-08 | docs/FDD.md | Dependência | Revisão de segurança ≥ 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| FDD-AC-01 | docs/FDD.md | Critério de Aceite | 1 linha PENDING por webhook inscrito, eventId compartilhado | TRANSCRICAO | [09:40] Bruno |
| FDD-AC-02 | docs/FDD.md | Critério de Aceite | Sem inscrito ⇒ nenhuma linha | TRANSCRICAO | [09:34] Bruno |
| FDD-AC-03 | docs/FDD.md | Critério de Aceite | Falha na outbox ⇒ status e history inalterados | TRANSCRICAO | [09:41] Diego |
| FDD-AC-04 | docs/FDD.md | Critério de Aceite | http:// ⇒ 400 com WEBHOOK_INVALID_URL | TRANSCRICAO | [09:23] Sofia |
| FDD-AC-05 | docs/FDD.md | Critério de Aceite | Secret só na criação; nunca em GET/PATCH | TRANSCRICAO | [09:31] Marcos |
| FDD-AC-06 | docs/FDD.md | Critério de Aceite | Headers enviados e assinatura válida | TRANSCRICAO | [09:44] Diego |
| FDD-AC-07 | docs/FDD.md | Critério de Aceite | Assinatura com nova e antiga por 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-AC-08 | docs/FDD.md | Critério de Aceite | Timeout de 10 s ⇒ primeiro intervalo da política confirmada | TRANSCRICAO | [09:42] Diego; [09:17] Larissa |
| FDD-AC-09 | docs/FDD.md | Critério Bloqueado | Limite de chamadas e DLQ dependem da decisão RFC-OQ-08 | TRANSCRICAO | [09:16] Diego; [09:17] Diego; [09:17] Larissa |
| FDD-AC-10 | docs/FDD.md | Critério de Aceite | Payload > 64 KB ⇒ DLQ sem HTTP | TRANSCRICAO | [09:24] Larissa |
| FDD-AC-11 | docs/FDD.md | Critério de Aceite | Replay: OPERATOR 403; ADMIN 202 com mesmo eventId e log | TRANSCRICAO | [09:36] Sofia |
| FDD-AC-12 | docs/FDD.md | Critério de Aceite | Segundo replay ⇒ 409 | TRANSCRICAO | [09:36] Sofia |
| FDD-AC-13 | docs/FDD.md | Critério de Aceite | Deliveries lista tentativas; pageSize ≤ 100 | TRANSCRICAO | [09:34] Marcos |
| FDD-AC-14 | docs/FDD.md | Critério de Aceite | API e worker independentes | TRANSCRICAO | [09:11] Diego |
| FDD-AC-15 | docs/FDD.md | Critério de Aceite | Testes existentes de pedidos e auth passam sem alteração | CODIGO | tests/orders.test.ts |
| FDD-AC-16 | docs/FDD.md | Critério de Aceite | Nenhum log contém secret | TRANSCRICAO | [09:22] Diego |
| FDD-RISK-01 | docs/FDD.md | Risco | Head-of-line blocking | TRANSCRICAO | [09:12] Diego; [09:42] Diego |
| FDD-RISK-02 | docs/FDD.md | Risco | Ordenação violada sob retry | TRANSCRICAO | [09:13] Larissa |
| FDD-RISK-03 | docs/FDD.md | Risco | Timeout da transação interativa com mais consultas | TRANSCRICAO | [09:04] Bruno |
| FDD-RISK-04 | docs/FDD.md | Risco | Secret vazada por log ou resposta | TRANSCRICAO | [09:22] Diego |
| FDD-RISK-05 | docs/FDD.md | Risco | Reenvio duplicado após crash | TRANSCRICAO | [09:24] Diego |
| FDD-RISK-06 | docs/FDD.md | Risco | Crescimento da outbox e deliveries sem arquivamento | TRANSCRICAO | [09:07] Bruno; [09:08] Diego |
| FDD-RISK-07 | docs/FDD.md | Risco | Redirect/URL interna como destino | TRANSCRICAO | [09:23] Sofia |
| FDD-RISK-08 | docs/FDD.md | Risco | Dois workers por engano quebram ordenação | TRANSCRICAO | [09:13] Diego |
| FDD-RISK-09 | docs/FDD.md | Risco de Segurança (derivado) | HTTPS não restringe destinos internos/privados; política pendente | TRANSCRICAO | [09:23] Sofia |
| FDD-RISK-10 | docs/FDD.md | Risco (derivado) | Política de remoção pode apagar histórico/outbox/DLQ | TRANSCRICAO | [09:08] Diego; [09:33] Bruno |
| FDD-RISK-11 | docs/FDD.md | Risco (derivado) | Acesso cross-customer não está definido por autenticação/role | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-PLAN-01 | docs/FDD.md | Plano | Sprint 1: modelagem de outbox e DLQ | TRANSCRICAO | [09:46] Larissa |
| FDD-PLAN-02 | docs/FDD.md | Plano | Sprint 2: worker e retry | TRANSCRICAO | [09:46] Larissa |
| FDD-PLAN-03 | docs/FDD.md | Plano | ½ sprint: CRUD de configuração e deliveries | TRANSCRICAO | [09:46] Larissa |
| FDD-PLAN-04 | docs/FDD.md | Plano | ½ sprint: integração no order.service e testes ponta a ponta | TRANSCRICAO | [09:46] Larissa |
| FDD-PLAN-05 | docs/FDD.md | Plano | HMAC, schemas, validações e revisão da Sofia no fim | TRANSCRICAO | [09:46] Larissa; [09:47] Larissa |

## PRD

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Ciclo de vida PENDING → PAID → PROCESSING → SHIPPED → DELIVERED / CANCELLED | CODIGO | src/modules/orders/order.status.ts |
| PRD-META-01 | docs/PRD.md | Metadado | Prazo alvo fim de novembro; 3 sprints | TRANSCRICAO | [09:45] Marcos; [09:47] Larissa |
| PRD-PROB-01 | docs/PRD.md | Problema | Pedido formal de Atlas, MaxDistribuição e Nova Cargo | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-02 | docs/PRD.md | Problema | Polling em GET /orders lento e caro | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-03 | docs/PRD.md | Problema | Risco de churn da Atlas até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-04 | docs/PRD.md | Requisito Não Funcional | Tempo real = abaixo de 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-PUB-01 | docs/PRD.md | Público-alvo | Clientes B2B integradores | TRANSCRICAO | [09:00] Marcos |
| PRD-PUB-02 | docs/PRD.md | Público-alvo | Usuários da plataforma que representam o cliente | TRANSCRICAO | [09:32] Marcos |
| PRD-PUB-03 | docs/PRD.md | Público-alvo | Administradores (ADMIN) | TRANSCRICAO | [09:36] Sofia |
| PRD-PUB-04 | docs/PRD.md | Público-alvo | PM / portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-CEN-01 | docs/PRD.md | Cenário | Atlas assina só SHIPPED e DELIVERED | TRANSCRICAO | [09:33] Marcos |
| PRD-CEN-02 | docs/PRD.md | Cenário | Manutenção planejada de 2 h coberta por retentativas | TRANSCRICAO | [09:16] Diego |
| PRD-CEN-03 | docs/PRD.md | Cenário | Vazamento de secret e rotação com 24 h | TRANSCRICAO | [09:21] Sofia; [09:22] Diego |
| PRD-CEN-04 | docs/PRD.md | Cenário | Notificação duplicada descartada pelo cliente | TRANSCRICAO | [09:24] Diego |
| PRD-CEN-05 | docs/PRD.md | Cenário | Cliente consulta histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-CEN-06 | docs/PRD.md | Cenário | Após limite de retry a decidir, falha definitiva e replay por ADMIN | TRANSCRICAO | [09:17] Marcos; [09:18] Diego |
| PRD-CEN-07 | docs/PRD.md | Cenário | Cliente com vários endpoints identifica o cadastro | TRANSCRICAO | [09:44] Sofia |
| PRD-OBJ-01 | docs/PRD.md | Objetivo / Métrica | Alvo < 10 s; percentil e perfil de carga pendentes | TRANSCRICAO | [09:02] Marcos; [09:10] Larissa |
| PRD-OBJ-02 | docs/PRD.md | Objetivo / Métrica | 3 de 3 clientes solicitantes em produção até fim de novembro | TRANSCRICAO | [09:45] Marcos; [09:00] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo / Métrica | 0 mudanças de status inscritas sem evento | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-04 | docs/PRD.md | Objetivo / Métrica Pendente | Janela de retry depende da contagem de chamadas ainda não decidida | TRANSCRICAO | [09:17] Diego; [09:17] Larissa |
| PRD-OBJ-05 | docs/PRD.md | Objetivo / Métrica | 100% das falhas definitivas registradas e reprocessáveis | TRANSCRICAO | [09:18] Diego |
| PRD-OBJ-06 | docs/PRD.md | Objetivo / Métrica | Redução do polling em GET /orders (sem meta numérica) | TRANSCRICAO | [09:00] Marcos |
| PRD-IN-01 | docs/PRD.md | Escopo | Notificação outbound | TRANSCRICAO | [09:02] Marcos |
| PRD-IN-02 | docs/PRD.md | Escopo | Gestão de webhooks pela API | TRANSCRICAO | [09:31] Marcos; [09:33] Bruno |
| PRD-IN-03 | docs/PRD.md | Escopo | Filtro por status | TRANSCRICAO | [09:33] Marcos |
| PRD-IN-04 | docs/PRD.md | Escopo | Assinatura e rotação de secret | TRANSCRICAO | [09:22] Sofia |
| PRD-IN-05 | docs/PRD.md | Escopo | Retentativas e fila de falhas com replay ADMIN | TRANSCRICAO | [09:17] Larissa; [09:19] Larissa |
| PRD-IN-06 | docs/PRD.md | Escopo | Histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-OUT-01 | docs/PRD.md | Fora de Escopo | Aviso por e-mail (adiado para próxima fase) | TRANSCRICAO | [09:37] Larissa; [09:38] Marcos |
| PRD-OUT-02 | docs/PRD.md | Fora de Escopo | Dashboard visual (descartado; time de frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-OUT-03 | docs/PRD.md | Fora de Escopo | Rate limiting de envio (adiado; observar) | TRANSCRICAO | [09:39] Larissa; [09:39] Diego |
| PRD-OUT-04 | docs/PRD.md | Fora de Escopo | Webhooks de entrada | TRANSCRICAO | [09:02] Marcos |
| PRD-OUT-05 | docs/PRD.md | Fora de Escopo | Arquivamento de entregues (~30 dias) | TRANSCRICAO | [09:08] Diego |
| PRD-OUT-06 | docs/PRD.md | Fora de Escopo | Ordenação global e múltiplos processadores | TRANSCRICAO | [09:13] Diego; [09:14] Marcos |
| PRD-OUT-07 | docs/PRD.md | Fora de Escopo | Exactly-once | TRANSCRICAO | [09:25] Diego |
| PRD-OUT-08 | docs/PRD.md | Fora de Escopo | Restrição de perfil para gerenciar webhooks (adiado) | TRANSCRICAO | [09:37] Sofia |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook com customerId no corpo, URL e status | TRANSCRICAO | [09:31] Marcos; [09:32] Larissa |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Listar webhooks de um cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Editar webhook (URL, status, ativo) | TRANSCRICAO | [09:33] Bruno; [09:21] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Remover webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Notificar só webhooks ativos que assinam o novo status | TRANSCRICAO | [09:33] Marcos; [09:34] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Conteúdo da notificação sem itens | TRANSCRICAO | [09:43] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Notificação reflete o pedido no momento da mudança | TRANSCRICAO | [09:52] Larissa |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Notificação assinada | TRANSCRICAO | [09:19] Sofia; [09:20] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Identificador do evento, do webhook e data/hora do envio | TRANSCRICAO | [09:44] Diego; [09:25] Diego; [09:44] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Rotacionar secret com 24 h de convivência | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Retentativas 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Fila de falhas definitivas com motivo e data/hora | TRANSCRICAO | [09:18] Diego; [09:15] Diego |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Replay manual por ADMIN com registro de quem executou | TRANSCRICAO | [09:36] Sofia; [09:18] Diego |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | Histórico de entregas (sucesso/falha, payload, resposta, tempo) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-16 | docs/PRD.md | Requisito Funcional | Recusar URLs não https | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Entrega < 10 s; verificação a cada 2 s | TRANSCRICAO | [09:02] Marcos; [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Não bloquear mudança de status por cliente lento | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Status mudou ⇒ evento registrado | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | At-least-once com duplicatas identificáveis | TRANSCRICAO | [09:24] Diego; [09:26] Larissa |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Processamento separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Ordem por pedido com processador único; sem ordem global | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional Pendente | Cinco delays listados; contagem total de chamadas precisa confirmação | TRANSCRICAO | [09:16] Diego; [09:17] Diego; [09:17] Larissa |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256 com secret única por webhook | TRANSCRICAO | [09:22] Sofia |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | TLS obrigatório | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Replay só ADMIN com auditoria; CRUD qualquer autenticado | TRANSCRICAO | [09:36] Sofia; [09:37] Sofia |
| PRD-NFR-12 | docs/PRD.md | Restrição | Payload ≤ 64 KB; acima disso erro, sem truncar | TRANSCRICAO | [09:24] Larissa; [09:23] Sofia |
| PRD-NFR-13 | docs/PRD.md | Requisito Não Funcional | Padrões do projeto, sem infraestrutura nova | TRANSCRICAO | [09:30] Larissa; [09:07] Diego |
| PRD-NFR-14 | docs/PRD.md | Restrição | Revisão de segurança ≥ 2 dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-DEC-01 | docs/PRD.md | Trade-off | Registro atômico e envio assíncrono | TRANSCRICAO | [09:06] Diego |
| PRD-DEC-02 | docs/PRD.md | Trade-off | 2 s de atraso mínimo aceito pelo PM | TRANSCRICAO | [09:10] Marcos |
| PRD-DEC-03 | docs/PRD.md | Trade-off | Backoff listado e DLQ; contagem/janela efetiva pendentes | TRANSCRICAO | [09:17] Diego; [09:17] Larissa |
| PRD-DEC-04 | docs/PRD.md | Trade-off | Mais gestão de secrets para conter vazamento | TRANSCRICAO | [09:21] Sofia |
| PRD-DEC-05 | docs/PRD.md | Trade-off | Cliente precisa deduplicar | TRANSCRICAO | [09:25] Sofia |
| PRD-DEC-06 | docs/PRD.md | Trade-off | Reuso de padrões para velocidade e consistência | TRANSCRICAO | [09:30] Larissa |
| PRD-DEC-07 | docs/PRD.md | Trade-off | Filtro e conteúdo congelados no momento do evento | TRANSCRICAO | [09:52] Larissa; [09:34] Bruno |
| PRD-DEP-01 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos; [09:40] Marcos |
| PRD-DEP-02 | docs/PRD.md | Dependência | Revisão de segurança da Sofia | TRANSCRICAO | [09:46] Sofia; [09:49] Sofia |
| PRD-DEP-03 | docs/PRD.md | Dependência | Alteração no fluxo de status (time de Pedidos) | TRANSCRICAO | [09:40] Bruno |
| PRD-DEP-04 | docs/PRD.md | Dependência | Novo processo de execução (Plataforma) | TRANSCRICAO | [09:11] Diego |
| PRD-DEP-05 | docs/PRD.md | Dependência | Clientes com https, HMAC e dedup | TRANSCRICAO | [09:23] Sofia; [09:24] Diego |
| PRD-DEP-06 | docs/PRD.md | Dependência | Confirmação do prazo com clientes | TRANSCRICAO | [09:47] Marcos; [09:49] Marcos |
| PRD-DEP-07 | docs/PRD.md | Dependência | Sessão de revisão do design | TRANSCRICAO | [09:50] Larissa |
| PRD-RISK-01 | docs/PRD.md | Risco | Perda da Atlas por atraso | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Cliente processa duplicata | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-03 | docs/PRD.md | Risco | Vazamento de secret | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Cliente lento atrasa outros clientes | TRANSCRICAO | [09:42] Diego; [09:13] Diego |
| PRD-RISK-05 | docs/PRD.md | Risco | Volume alto para um cliente (50/min) | TRANSCRICAO | [09:38] Diego |
| PRD-RISK-06 | docs/PRD.md | Risco | Falhas definitivas sem aviso ao cliente | TRANSCRICAO | [09:37] Larissa |
| PRD-RISK-07 | docs/PRD.md | Risco (derivado) | Relação de ownership entre usuário autenticado e customer não definida | TRANSCRICAO | [09:32] Larissa; [09:32] Bruno; [09:37] Sofia |
| PRD-AC-01 | docs/PRD.md | Critério de Aceite | Cadastro https devolve secret | TRANSCRICAO | [09:31] Marcos |
| PRD-AC-02 | docs/PRD.md | Critério de Aceite | URL http recusada | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-03 | docs/PRD.md | Critério de Aceite Pendente | Alvo < 10 s depende de perfil de carga/critério estatístico RFC-OQ-09 | TRANSCRICAO | [09:02] Marcos; [09:09] Diego |
| PRD-AC-04 | docs/PRD.md | Critério de Aceite | Status não assinado não gera notificação | TRANSCRICAO | [09:34] Bruno |
| PRD-AC-05 | docs/PRD.md | Critério de Aceite | Falha no registro mantém status anterior | TRANSCRICAO | [09:40] Bruno |
| PRD-AC-06 | docs/PRD.md | Critério de Aceite Pendente | Calendário de retry definido; número de chamadas à DLQ depende RFC-OQ-08 | TRANSCRICAO | [09:17] Larissa |
| PRD-AC-07 | docs/PRD.md | Critério de Aceite | Replay negado a OPERATOR, permitido a ADMIN com registro | TRANSCRICAO | [09:36] Sofia |
| PRD-AC-08 | docs/PRD.md | Critério de Aceite | Rotação com 24 h de convivência | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-09 | docs/PRD.md | Critério de Aceite | Histórico de entregas completo | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-10 | docs/PRD.md | Critério de Aceite | Corpo com campos definidos e sem itens | TRANSCRICAO | [09:43] Diego |
| PRD-AC-11 | docs/PRD.md | Critério de Aceite | Endpoints de pedidos inalterados | CODIGO | src/modules/orders/order.routes.ts |
| PRD-TEST-01 | docs/PRD.md | Estratégia de Teste | Unitários: assinatura, retentativas, limite, filtro (Vitest) | CODIGO | vitest.config.ts |
| PRD-TEST-02 | docs/PRD.md | Estratégia de Teste | Integração da API no padrão existente (Vitest + Supertest) | CODIGO | tests/orders.test.ts |
| PRD-TEST-03 | docs/PRD.md | Estratégia de Teste | Integração do fluxo de pedidos contra MySQL local | CODIGO | docker-compose.yml |
| PRD-TEST-04 | docs/PRD.md | Estratégia de Teste Pendente | E2E e alvo < 10 s sob perfil de carga a definir; retry após RFC-OQ-08 | TRANSCRICAO | [09:46] Larissa; [09:02] Marcos |
| PRD-TEST-05 | docs/PRD.md | Estratégia de Teste | Revisão de segurança manual | TRANSCRICAO | [09:46] Sofia |
| PRD-TEST-06 | docs/PRD.md | Estratégia de Teste | Acompanhamento das metas pós go-live; e-mail avaliado após medir | TRANSCRICAO | [09:37] Larissa |

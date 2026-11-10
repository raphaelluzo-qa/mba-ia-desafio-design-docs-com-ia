# ADR-002 — Worker em processo separado, lendo a outbox por polling de 2 segundos

## Status

**Aceito**

- **Data da decisão:** reunião técnica de quinta-feira, 09:10–09:13 (data civil não informada na transcrição)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)
- **Consultados:** Marcos (PM)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Contexto

Com a outbox definida ([ADR-001](ADR-001-outbox-no-mysql.md)), faltava decidir **quem** lê a tabela, **como** a leitura é disparada e **onde** esse código roda.

- O requisito de latência do cliente é "abaixo de 10 segundos" ([09:02] Marcos).
- O banco é MySQL (`provider = "mysql"` em `prisma/schema.prisma`). Ele não tem um mecanismo de notificação para processos externos equivalente ao `LISTEN/NOTIFY` do Postgres. Triggers só executam SQL ([09:09] Diego).
- A API sobe por `src/server.ts` (`npm run dev` / `npm start`). Se o worker rodasse dentro dela, qualquer restart da API interromperia as entregas ([09:11] Diego).
- Os clientes não pedem ordenação global, só querem saber se cada pedido mudou ([09:14] Marcos).

## Decisão

1. **Polling em loop a cada 2 segundos.** A cada ciclo, o worker busca os eventos pendentes mais antigos em lote pequeno, processa e marca o resultado ([09:09] Diego, [09:08] Diego). Os 2 s são o intervalo de polling, não uma garantia de latência total; tempo de processamento, lote, carga e serialização também influem. A capacidade para cumprir o alvo <10 s precisa ser validada (RFC-OQ-09).
2. **Processo separado da API.** Nova entry point `src/worker.ts`, no mesmo formato de `src/server.ts`, com um script `npm run worker` ([09:11] Larissa). A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.worker.ts` ([09:28] Bruno).
3. **Mesmo banco, mesma stack, instância própria de `PrismaClient`.** O `PrismaClient` é por processo. O worker usa a mesma `DATABASE_URL` e cria sua instância com o `createPrismaClient()` de `src/config/database.ts` ([09:11] Bruno, [09:30] Bruno).
4. **Worker único (single-worker).** A ordem de processamento segue `created_at` da outbox, o que dá ordenação implícita **por `order_id`** enquanto houver um único worker ([09:12] Diego). Isso é uma limitação conhecida, não uma garantia de ordenação global ([09:13] Larissa).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Trigger no banco para acordar o worker (mais reativo)** | O MySQL não tem listener nativo. Avisar o worker exigiria improvisos, como escrever em arquivo ou chamar um endpoint ([09:09] Bruno propôs, [09:09] Diego descartou). O polling de 2 s já atende o requisito. |
| **Worker dentro do processo da API** | Um restart da API derruba o worker junto ([09:11] Diego). O worker também passaria a competir por recursos com o tráfego HTTP. |
| **Múltiplos workers em paralelo** | Perde a garantia de ordem por pedido. Particionar por `order_id` ou usar lock pessimista resolveria, mas foi classificado como "problema do futuro" ([09:13] Diego). Fica como questão em aberto no RFC. |

## Consequências

**Positivas**
- É simples de implementar e de operar, sem dependência nova.
- Restarts e deploys da API não afetam as entregas, e vice-versa.
- A ordem de entrega por pedido é preservada no fluxo normal.

**Negativas**
- O banco recebe uma consulta a cada 2 s mesmo sem eventos. Os índices da outbox mitigam o custo.
- Existe latência mínima de até ~2 s. Foi aceita explicitamente ([09:10] Larissa, [09:10] Marcos).
- **Throughput limitado e head-of-line blocking:** um worker único processa em sequência. Um endpoint que só falha por timeout (10 s) atrasa os eventos seguintes do lote.
- **Ordenação sob retry:** quando um evento entra em backoff ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)), um evento posterior do mesmo pedido pode ser entregue antes dele. O cliente precisa usar o campo `timestamp` do payload. Isso está registrado como questão em aberto no RFC.
- É um novo processo para implantar e monitorar.

**Trade-off explícito:** trocamos reatividade e escala horizontal por simplicidade operacional e ordenação por pedido. Os 2 s de latência cabem folgados na meta de 10 s.

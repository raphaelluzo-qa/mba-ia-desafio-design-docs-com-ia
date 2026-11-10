# ADR-001 — Padrão Outbox no MySQL existente para eventos de webhook

## Status

**Aceito**

- **Data da decisão:** reunião técnica de quinta-feira, 09:08 (data civil não informada na transcrição)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)
- **Consultados:** Marcos (PM), Sofia (Segurança)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-snapshot-do-payload-e-filtro-na-insercao.md)

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pediram para ser notificados quando o status dos pedidos muda, em vez de ficar consultando `GET /orders` periodicamente ([09:00] Marcos).

A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de uma transação interativa do Prisma (`this.prisma.$transaction(async (tx) => ...)`). Essa transação já:

1. valida a transição com `canTransition` (`src/modules/orders/order.status.ts`);
2. debita ou repõe estoque (`debitStock` / `replenishStock`);
3. atualiza `orders.status`;
4. insere uma linha em `order_status_history`.

O time precisava decidir como a notificação sai do sistema sem deixar a transação mais lenta e sem criar inconsistência entre "o status mudou" e "o evento foi registrado". Forças em jogo:

- Uma chamada HTTP dentro da transação deixaria a mudança de status de todos os pedidos refém do cliente mais lento ([09:04] Bruno).
- Se o cliente estiver fora do ar, não faz sentido desfazer a mudança de status ([09:04] Bruno).
- O time é pequeno e não quer operar infraestrutura nova ([09:07] Diego).
- Não pode haver mudança de status sem evento correspondente ([09:40] Bruno, [09:41] Diego).

## Decisão

Adotar o **padrão Transactional Outbox no MySQL que a aplicação já usa**:

- Criar a tabela `webhook_outbox`. Dentro da **mesma transação** de `changeStatus`, depois do insert em `order_status_history`, inserir as linhas de evento ([09:06] Diego).
- A inserção é feita por uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o `tx` da transação corrente. O `OrderService` não recebe um repository de webhooks injetado ([09:41] Bruno, [09:41] Diego).
- Se o insert na outbox falhar, a transação inteira faz rollback: o status não muda ([09:40] Bruno).
- A outbox tem índice por status (pendente, processando, falhou, entregue) e por `created_at` ([09:08] Diego).
- As chaves primárias são UUID, como no resto do schema (`@default(uuid()) @db.Char(36)` em `prisma/schema.prisma`) ([09:51] Larissa).
- O envio HTTP fica **fora** da transação, com um worker dedicado ([ADR-002](ADR-002-worker-separado-em-polling.md)).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Disparo HTTP síncrono dentro de `changeStatus`** | A transação já é pesada. Um cliente lento travaria mudanças de status de outros pedidos, e uma indisponibilidade do cliente não justifica rollback do status ([09:04] Bruno, [09:06] Diego: "Síncrono está fora de questão"). |
| **Redis Streams / fila externa** | Exige subir e operar infraestrutura nova (Redis Cluster). Para um time pequeno, isso é overengineering ([09:07] Larissa, [09:07] Diego). Também perderia a atomicidade com a transação SQL sem um mecanismo adicional. |
| **Injetar um repository de webhooks no `OrderService`** | Funciona, mas acopla mais os módulos. Uma função que recebe o `tx` resolve com menos superfície ([09:41] Diego). |

## Consequências

**Positivas**
- **Atomicidade:** se a transação commitou, o evento existe; se fez rollback, o evento some junto ([09:06] Diego).
- Nenhuma infraestrutura nova: mesmo banco, mesmo Prisma, mesmo `DATABASE_URL`.
- A mudança de status não depende da disponibilidade do cliente.
- A outbox serve como trilha consultável para debug e para o histórico de entregas.

**Negativas**
- A transação de `changeStatus` ganha mais uma leitura (webhooks ativos do customer) e N inserts. Ela continua sujeita ao timeout padrão da transação interativa do Prisma.
- Uma falha na outbox **bloqueia** a mudança de status. É o preço da garantia "não existe status mudado sem evento".
- A tabela cresce continuamente. O arquivamento das linhas entregues (~30 dias) ficou fora do escopo desta feature ([09:08] Diego).
- A latência mínima de entrega passa a depender do intervalo de leitura do worker ([ADR-002](ADR-002-worker-separado-em-polling.md)).

**Trade-off explícito:** aceitamos acoplar a mudança de status à saúde da tabela de outbox, e uma latência de segundos, em troca de consistência forte entre status e evento sem nova infraestrutura.

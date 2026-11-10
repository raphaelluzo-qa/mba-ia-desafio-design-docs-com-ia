# RFC-001 — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
|---|---|
| **RFC** | 001 |
| **Título** | Sistema de Webhooks de Notificação de Pedidos (outbound) |
| **Autor** | @marcos-r-r (redação assistida por IA, a partir da reunião técnica conduzida por Larissa) |
| **Status** | Em revisão |
| **Data da reunião de origem** | Não informada na transcrição (quinta-feira, 09:00–09:53) |
| **Revisores** (RFC-META-01) | Larissa (Tech Lead), Marcos (Product Manager), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Engenharia de Segurança) |
| **Documentos do pacote** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

---

## 1. Resumo executivo (TL;DR)

Propomos **webhooks outbound** para avisar clientes B2B quando o status de um pedido muda, com alvo de latência **abaixo de 10 s** para endpoint saudável. O perfil de carga e o critério estatístico para validar esse alvo ainda precisam ser definidos.

- **Registro do evento:** entra numa **tabela outbox no MySQL**, na mesma transação de `OrderService.changeStatus`. Status mudado sem evento registrado é impossível.
- **Entrega:** feita por um **worker em processo separado**, que lê a outbox **por polling a cada 2 s**.
- **Assinatura:** cada entrega vai assinada com **HMAC-SHA256** usando uma **secret por endpoint**, rotacionável com grace period de 24 h.
- **Garantia:** **at-least-once**, com `X-Event-Id` para o cliente deduplicar.
- **Falhas:** retentativas com os intervalos discutidos (**1m/5m/30m/2h/12h**) e depois uma **DLQ** com replay manual por ADMIN; a contagem total de chamadas ainda está pendente (RFC-OQ-08).
- **Código:** o módulo segue os **padrões existentes** do projeto.

Sem infraestrutura nova. Estimativa de **3 sprints**, incluindo a revisão de segurança.

## 2. Contexto e problema

- **RFC-CTX-01.** Atlas Comercial, MaxDistribuição e Nova Cargo fazem polling em `GET /orders` para detectar mudanças. A integração fica lenta e cara para eles. A Atlas sinalizou que pode migrar para um concorrente se isso não for entregue até o fim do trimestre ([09:00] Marcos).
- **RFC-CTX-02.** Para esses clientes, "tempo real" é **qualquer coisa abaixo de 10 s** ([09:02] Marcos).
- **RFC-CTX-03.** Hoje o OMS **não tem nenhum mecanismo de notificação, evento ou fila**. A única rota relacionada é `PATCH /api/v1/orders/:id/status` (`src/modules/orders/order.routes.ts`).
- **RFC-CTX-04.** A mudança de status já é uma transação pesada: atualiza `orders`, insere em `order_status_history` e mexe em `stock_quantity` (`src/modules/orders/order.service.ts`, [09:04] Bruno). Qualquer solução precisa **não degradar** esse caminho e **não perder eventos**.

## 3. Proposta técnica

### 3.1 Visão geral

```
 PATCH /orders/:id/status
          │
          ▼
 ┌─────────────────── transação MySQL (changeStatus) ───────────────────┐
 │ update orders → insert order_status_history → publishWebhookEvent(tx)│
 │                                     └─► insert webhook_outbox (N)    │
 └──────────────────────────────────────────────────────────────────────┘
                         │ commit
                         ▼
           ┌─────────────────────────────┐   POST https://cliente/...
           │ src/worker.ts (processo     │   X-Event-Id, X-Webhook-Id,
           │ separado, polling 2 s)      │── X-Signature (HMAC), X-Timestamp
           └─────────────────────────────┘
                 │ falha → backoff 1m/5m/30m/2h/12h
                 ▼ esgotou
           webhook_dead_letter ──► POST /admin/webhooks/dead-letter/:id/replay (ADMIN)
```

### 3.2 Componentes propostos

| ID | Componente | Responsabilidade | Decisão |
|---|---|---|---|
| RFC-PROP-01 | **Outbox (`webhook_outbox`)** | Registra eventos na mesma transação da mudança de status | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| RFC-PROP-02 | **Publisher (`publishWebhookEvent`)** | Recebe o `tx`, filtra os webhooks inscritos e grava o snapshot do payload | [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md) |
| RFC-PROP-03 | **Worker (`src/worker.ts`)** | Processo separado, polling de 2 s, envio HTTP com timeout de 10 s, single-worker | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| RFC-PROP-04 | **Retry + DLQ (`webhook_dead_letter`)** | Intervalos de backoff discutidos, replay manual por ADMIN; limite de chamadas/DLQ pendente em RFC-OQ-08 | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| RFC-PROP-05 | **Segurança** | HMAC-SHA256 do corpo, secret por endpoint, rotação com grace period de 24 h, `https` obrigatório | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| RFC-PROP-06 | **Semântica de entrega** | At-least-once, `X-Event-Id` estável entre tentativas | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| RFC-PROP-07 | **API de configuração** | CRUD de webhooks, rotação de secret, histórico de entregas e replay de DLQ | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) (padrões); contratos no [FDD §6](FDD.md#6-contratos-públicos) |

### 3.3 Princípios que guiam a proposta

1. **RFC-PRINC-01 — Consistência antes de latência.** Aceitamos 2 s de latência mínima para ter atomicidade entre status e evento ([09:10] Larissa).
2. **RFC-PRINC-02 — Zero infraestrutura nova.** Mesmo MySQL, mesmo Prisma, mesmo Pino ([09:07] Diego, [09:29] Bruno).
3. **RFC-PRINC-03 — Cada endpoint é isolado:** secret própria ([09:21] Sofia), filtro próprio ([09:33] Bruno) e identificação própria no envio ([09:44] Sofia).
4. **RFC-PRINC-04 — O webhook é um módulo como os outros** (`src/modules/webhooks`) ([09:30] Larissa).

## 4. Alternativas consideradas

Cada alternativa abaixo foi colocada na mesa e descartada na reunião.

| ID | Alternativa | Trade-off que levou ao descarte | Fonte |
|---|---|---|---|
| RFC-ALT-01 | **Disparo HTTP síncrono dentro de `changeStatus`** | Um cliente lento trava a mudança de status de outros pedidos, e uma indisponibilidade dele forçaria rollback de uma mudança de negócio válida. A latência zero não compensa acoplar nossa transação à disponibilidade de terceiros. | [09:04] Bruno, [09:06] Diego |
| RFC-ALT-02 | **Redis Streams / fila externa** | Seria mais reativo e escalável, mas exige subir e operar infraestrutura (Redis Cluster). Para um time pequeno, é overengineering, e ainda perde a atomicidade nativa com a transação SQL. | [09:07] Larissa, [09:07] Diego |
| RFC-ALT-03 | **Trigger do banco para notificar o worker** | Seria mais reativo que polling, mas o MySQL não tem `LISTEN/NOTIFY`. O aviso ao processo externo exigiria improviso (arquivo, endpoint). O polling de 2 s já cumpre a meta de 10 s. | [09:09] Bruno, [09:09] Diego |
| RFC-ALT-04 | **Worker no mesmo processo da API** | É menos coisa para implantar, mas um restart da API derruba as entregas. | [09:11] Diego |
| RFC-ALT-05 | **Retry indefinido** ou **só 3 tentativas** | O indefinido deixa evento pendurado para sempre. Três tentativas cobrem só ~30 min e matam o evento numa manutenção de 2 h. | [09:15] Diego, [09:16] Bruno, [09:16] Diego |
| RFC-ALT-06 | **Exactly-once** | Exige coordenação dos dois lados, com complexidade desproporcional. At-least-once com `event_id` resolve a grande maioria dos casos. | [09:25] Diego |
| RFC-ALT-07 | **Secret global da plataforma** | É mais simples de gerenciar, mas um vazamento compromete todos os clientes. | [09:21] Sofia |

Alternativas menores (DLQ como status na outbox, filtro no envio, render no envio) estão registradas nos ADRs [003](adrs/ADR-003-retry-com-backoff-e-dlq.md) e [007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md).

## 5. Questões em aberto

| ID | Questão | Estado na reunião | Próximo passo | Fonte |
|---|---|---|---|---|
| RFC-OQ-01 | **Rate limiting de saída por cliente.** 50 mudanças em 1 min geram 50 chamadas. | Adiado: "observar e decidir depois" | Acompanhar volume por webhook nas métricas do worker | [09:38] Diego, [09:39] Larissa |
| RFC-OQ-02 | **Aviso ao cliente quando o webhook falha repetidamente (e-mail).** | Adiado para a próxima fase, depois de medir impacto | Reavaliar com dados de DLQ após o go-live | [09:37] Marcos, [09:37] Larissa |
| RFC-OQ-03 | **Escala para múltiplos workers.** Particionar por `order_id` ou usar lock pessimista. | Adiado: "problema do futuro" | Gatilho: idade do evento pendente mais antigo acima da meta de 10 s | [09:13] Bruno, [09:13] Diego |
| RFC-OQ-04 | **Arquivamento das linhas entregues da outbox (~30 dias).** | Fora do escopo desta feature | Definir job e retenção em feature própria | [09:08] Diego |
| RFC-OQ-05 | **Endurecer a autorização do CRUD de webhooks** (hoje qualquer role autenticada). | Adiado: "mais pra frente" | Rever depois do go-live, junto com o modelo de usuários que representam clientes | [09:37] Sofia |
| RFC-OQ-06 | **Ordenação por pedido durante retry.** Um evento em backoff pode ser ultrapassado por um evento posterior do mesmo pedido. | Não discutido: surgiu da análise da garantia "ordering por `order_id` enquanto single-worker" | Levar à sessão de revisão do design: aceitar e documentar (cliente usa `timestamp`) ou bloquear por `(webhook, order_id)` | [09:12] Diego, [09:50] Larissa |
| RFC-OQ-07 | **Detalhes de segurança para a revisão:** formato do `X-Signature` com duas assinaturas no grace period, proteção em repouso da secret (precisa ser recuperável para o HMAC) e se `X-Timestamp` entra na base assinada. | Não detalhado. Sofia exigiu revisão de HMAC e geração de secret | Revisão de segurança, com pelo menos 2 dias úteis antes do deploy | [09:46] Sofia |
| RFC-OQ-08 | **Contagem de tentativas.** A reunião diz “5 tentativas” e lista cinco intervalos, mas não define se o envio inicial está incluído na contagem. | Não decidido; a interpretação de 1 envio + 5 retries não deve ser tratada como aprovada | Confirmar quantidade total de chamadas, aplicação do último intervalo e momento de mover para DLQ na revisão do design | [09:16] Diego, [09:17] Diego, [09:17] Larissa |
| RFC-OQ-09 | **Critério de desempenho sob carga.** O objetivo é ficar abaixo de 10 s; o desenho proposto usa polling de 2 s e worker único/sequencial, sem carga, concorrência ou percentil definidos. | Limite de produto discutido; capacidade e método de validação não discutidos | Definir perfil de carga, métrica estatística e condição de aprovação antes de afirmar que a arquitetura atende o alvo; reavaliar escala se necessário | [09:02] Marcos, [09:09] Diego, [09:12] Diego |
| RFC-OQ-10 | **Autorização entre usuários e clientes.** O `customerId` é informado na API e o CRUD pode ser usado por qualquer role autenticada, mas não há regra registrada ligando usuário a customer. | A reunião define autenticação e papéis, mas não decide ownership/isolamento por customer | Confirmar com produto/segurança como autorizar acesso a cada customer antes de implementar CRUD, leitura e alterações | [09:32] Larissa, [09:32] Bruno, [09:37] Sofia |
| RFC-OQ-11 | **Política de destino para evitar acesso a endereços internos.** `https` e redirects manuais não estabelecem se destinos loopback, privados, link-local ou mudanças de DNS são permitidos. | Lacuna identificada na revisão; a reunião só exige esquema `https` | Segurança deve definir a política de destino e sua validação antes de considerar mitigado o risco de URL interna; não inferir que HTTPS resolve esse risco | [09:23] Sofia |
| RFC-OQ-12 | **Exclusão e retenção dos dados de webhook.** A reunião pede operação de remoção, mas não define hard delete versus desativação nem o efeito em outbox, histórico de deliveries e DLQ. | Comportamento de persistência não decidido | Decidir semântica de exclusão e retenção antes de fechar contrato e migração; o FDD deixa as ações referenciais em aberto | [09:08] Diego, [09:33] Bruno |

## 6. Impacto e riscos

### 6.1 Impacto

| ID | Área | Impacto |
|---|---|---|
| RFC-IMP-01 | **Caminho crítico de pedidos** | `changeStatus` ganha uma leitura (webhooks ativos do customer) e N inserts na mesma transação. Uma falha nesses inserts faz rollback da mudança de status ([09:40] Bruno). |
| RFC-IMP-02 | **Banco de dados** | 4 tabelas novas (configuração, outbox, histórico de entregas e DLQ) e um polling a cada 2 s. |
| RFC-IMP-03 | **Operação** | Um novo processo (`npm run worker`) para implantar e monitorar ([09:11] Larissa). |
| RFC-IMP-04 | **Clientes** | Precisam validar HMAC, deduplicar por `X-Event-Id` e aceitar `https` obrigatório. A documentação fica no portal do desenvolvedor ([09:26] Marcos, [09:40] Marcos). |
| RFC-IMP-05 | **Segurança** | Secrets de clientes passam a ser armazenadas pela plataforma. A revisão da Sofia é obrigatória ([09:46] Sofia). |
| RFC-IMP-06 | **Prazo** | 3 sprints com a revisão de segurança incluída; a Atlas espera até o fim de novembro ([09:45] Marcos, [09:47] Larissa). |

### 6.2 Riscos

| ID | Risco | Mitigação |
|---|---|---|
| RFC-RISK-01 | **Head-of-line blocking no worker único.** Endpoints que só dão timeout (10 s) atrasam os demais clientes e podem estourar o alvo de 10 s. | Timeout de 10 s, medição de latência/idade da fila; perfil de carga e critério de validação pendentes (RFC-OQ-03, RFC-OQ-09) |
| RFC-RISK-06 | **Acesso cruzado entre customers:** autenticação sem regra de ownership não demonstra isolamento de dados de configuração e deliveries. | Definir autorização por customer antes de implementar; não presumir que `authenticate` ou role resolva a relação usuário-cliente (RFC-OQ-10) |
| RFC-RISK-07 | **SSRF/destino interno:** exigir HTTPS e não seguir redirects limita alguns caminhos, mas não define restrições a destinos privados ou DNS rebinding. | Política de destino e validação sujeitas à revisão de segurança; item em aberto RFC-OQ-11 |
| RFC-RISK-08 | **Perda de histórico/entregas pendentes ao remover configuração** se a política futura excluir dados associados. | Não fechar o comportamento de DELETE nem migration até decidir retenção e exclusão em RFC-OQ-12 |
| RFC-RISK-02 | **Cliente não deduplica** e processa o mesmo evento duas vezes. | `X-Event-Id` estável e documentação destacada no portal ([09:26] Marcos) |
| RFC-RISK-03 | **Vazamento de secret** do lado do cliente (já aconteceu). | Secret por endpoint, rotação com 24 h de grace e `redact` no logger ([09:22] Diego) |
| RFC-RISK-04 | **A outbox bloquear mudanças de status** (tabela ou índice com problema). | Insert simples e indexado, testes de integração no `changeStatus`, alerta de erro na transação |
| RFC-RISK-05 | **Prazo com risco de churn da Atlas.** | Plano em 3 sprints, escopo enxuto (sem e-mail, sem dashboard, sem rate limit) e revisão de segurança já reservada |

## 7. Decisões relacionadas

| ADR | Decisão |
|---|---|
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL existente |
| [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) | Worker em processo separado, polling de 2 s, single-worker |
| [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) | Retry com backoff 1m/5m/30m/2h/12h e DLQ em tabela separada |
| [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256, secret por endpoint e rotação com grace de 24 h |
| [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) | At-least-once com `X-Event-Id` |
| [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) | Reuso dos padrões existentes do projeto |
| [ADR-007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md) | Snapshot do payload e filtro de eventos na inserção |

**Próximo passo (RFC-NEXT-01):** sessão de revisão com Bruno e Diego antes de começar a codar ([09:50] Larissa). As questões RFC-OQ-06, RFC-OQ-08, RFC-OQ-09, RFC-OQ-10 e RFC-OQ-12 precisam ser resolvidas antes de fechar implementação; RFC-OQ-07 e RFC-OQ-11 exigem revisão de segurança.

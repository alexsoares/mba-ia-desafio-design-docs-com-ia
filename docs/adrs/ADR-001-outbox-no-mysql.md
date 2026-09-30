# ADR-001 — Padrão Transactional Outbox no MySQL existente

- **Status:** Aceito
- **Data:** reunião técnica de kickoff (quinta-feira, 09:00)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Consultados:** Marcos (PM), Sofia (Segurança)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-payload-snapshot-na-insercao.md)

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) precisam ser notificados quando o status de seus pedidos muda, com latência percebida inferior a 10 segundos [09:00–09:02 Marcos].

A mudança de status já é uma operação transacional pesada. Em `src/modules/orders/order.service.ts`, o método `changeStatus` roda em `this.prisma.$transaction(async (tx) => …)` e, na mesma transação, atualiza `orders`, insere em `order_status_history` e debita ou repõe `stock_quantity` (regras em `src/modules/orders/order.status.ts`: `shouldDebitStock` / `shouldReplenishStock`) [09:04 Bruno].

Duas forças conflitam:

1. **Consistência:** se o status mudou, o evento **tem** que existir; se a transação fez rollback, o evento **não pode** existir [09:06 Diego, 09:40 Bruno].
2. **Isolamento de falhas:** um cliente lento ou fora do ar não pode travar nem desfazer a mudança de status de nenhum pedido [09:04 Bruno].

O time é pequeno, e hoje a infraestrutura se resume a um MySQL 8 (`docker-compose.yml`) acessado via Prisma (`prisma/schema.prisma`). Não existe fila nem broker [09:07 Diego].

## Decisão

Adotar o **padrão Transactional Outbox no próprio MySQL**:

- Criar a tabela `webhook_outbox`. Dentro da **mesma transação** do `changeStatus`, inserir uma linha por evento a entregar, logo após o insert em `order_status_history`.
- Se o insert na outbox falhar, a transação inteira faz rollback: não existe status alterado sem evento [09:40 Bruno].
- A inserção é feita por uma função pura que recebe o client da transação: `publishWebhookEvent(tx, order, fromStatus, toStatus)`. Assim o `OrderService` não precisa receber um repository inteiro por injeção [09:41 Bruno, 09:41 Diego].
- A tabela tem índices em `status` (`PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`) e em `created_at` [09:08 Diego].
- As chaves primárias são UUID (`@db.Char(36)`), seguindo o padrão de todos os modelos do `schema.prisma` [09:51 Larissa].
- O filtro de eventos é aplicado **na inserção**: se nenhum webhook ativo do cliente assina o `to_status`, nenhuma linha é criada [09:34 Bruno, 09:34 Diego].
- Um processo separado (ver [ADR-002](ADR-002-worker-separado-em-polling.md)) consome a tabela e faz as chamadas HTTP.

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Disparo HTTP síncrono dentro do `changeStatus`** | Um cliente lento seguraria a transação e travaria mudanças de status de outros pedidos. Com o cliente fora do ar, restaria desfazer a mudança de status, o que é inaceitável ("Não dá") [09:04 Bruno, 09:06 Diego]. |
| **Redis Streams (ou broker equivalente)** | Exige subir e operar mais infraestrutura (Redis Cluster). Para um time pequeno foi considerado overengineering [09:07 Larissa, 09:07 Diego]. Também não resolve sozinho a atomicidade entre o commit no MySQL e a publicação no broker. |
| **Publicar após o commit (fora da transação)** | Se o processo cair entre o commit e a publicação, o evento se perde: "perde a garantia toda" [09:41 Diego]. |

## Consequências

**Positivas**
- Atomicidade total entre mudança de status e registro do evento, sem transação distribuída [09:06 Diego].
- Nenhuma infraestrutura nova: mesmo banco, mesmo Prisma, mesmas migrations [09:07 Diego].
- A indisponibilidade de clientes fica totalmente desacoplada do fluxo de pedidos.
- A outbox serve como trilha persistente de tudo o que foi (ou deveria ter sido) enviado.

**Negativas / trade-offs**
- **Latência mínima igual ao intervalo de polling** (2s no pior caso), em vez de push imediato. Foi aceita explicitamente [09:10 Larissa].
- A transação de `changeStatus` ganha mais um `SELECT` (webhooks assinantes) e N `INSERT`s, o que aumenta levemente seu tempo.
- A tabela cresce continuamente. O arquivamento das linhas entregues (~30 dias) foi mencionado, mas ficou **fora do escopo** desta feature [09:08 Diego] e passa a ser dívida conhecida.
- Throughput limitado pelo polling de um único worker. Escalar exige particionamento ou lock (ver [ADR-002](ADR-002-worker-separado-em-polling.md)).

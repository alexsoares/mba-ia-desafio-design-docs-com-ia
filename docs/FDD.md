# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Status** | Pronto para implementação (pendente sessão de revisão Larissa/Bruno/Diego [09:50 Larissa] e revisão de segurança [09:46 Sofia]) |
| **Documentos base** | [PRD](PRD.md) · [RFC](RFC.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |
| **Stack** | Node.js ≥ 20 · TypeScript · Express 4 · Prisma 5.22 · MySQL 8 · Zod · Pino · Vitest (`package.json`, `docker-compose.yml`) |

> Convenção: os itens deste documento têm IDs (`FDD-…`) rastreados em [TRACKER.md](TRACKER.md). Onde a reunião não fixou um valor e o FDD precisou escolher um, o item está marcado como **(escolha de implementação)**.

---

## 1. Contexto e motivação técnica

O OMS não tem mecanismo de notificação externa. Os clientes B2B fazem polling em `GET /api/v1/orders` para descobrir mudanças de status [09:00 Marcos].

A mudança de status acontece em um único lugar, `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de `prisma.$transaction`. Nessa transação:

- lê o pedido com os itens;
- valida a transição (`canTransition`, em `src/modules/orders/order.status.ts`);
- debita ou repõe estoque;
- atualiza `orders.status`;
- insere em `order_status_history`.

Esse é o **único ponto de gancho** necessário para publicar eventos [09:40 Bruno].

Restrições técnicas que moldam o desenho:

- nenhuma chamada HTTP pode acontecer dentro dessa transação [09:04 Bruno];
- o evento precisa ser atômico com a mudança de status [09:06 Diego, 09:40 Bruno];
- nenhuma infraestrutura nova [09:07 Diego];
- reuso dos padrões do projeto [09:30 Larissa].

Decisões fechadas: [ADR-001](adrs/ADR-001-outbox-no-mysql.md) a [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md).

## 2. Objetivos técnicos

| ID | Objetivo |
|---|---|
| FDD-OBJ-01 | Entregar o evento ao cliente em **< 10s** após o commit da mudança de status quando o endpoint do cliente está saudável (polling de 2s + timeout de 10s como teto de uma tentativa) [09:02 Marcos, 09:09 Diego]. |
| FDD-OBJ-02 | **Zero divergência** entre status alterado e evento registrado: o insert na outbox participa da transação do `changeStatus` [09:40 Bruno]. |
| FDD-OBJ-03 | **Zero impacto** de clientes lentos ou offline na API de pedidos: nenhum I/O externo no processo da API [09:04 Bruno, 09:11 Diego]. |
| FDD-OBJ-04 | Entrega **at-least-once** com `X-Event-Id` estável em retentativas e replays [09:24 Diego, 09:25 Diego]. |
| FDD-OBJ-05 | Autenticidade e integridade verificáveis pelo cliente via HMAC-SHA256 com secret por endpoint [09:22 Sofia]. |
| FDD-OBJ-06 | Nenhuma dependência npm nova: HTTP via `fetch` nativo, HMAC e secret via `node:crypto` (`engines.node >= 20` em `package.json`) [09:29 Bruno, 09:30 Larissa]. |

## 3. Escopo e exclusões

**Dentro do escopo**
- Modelo de dados: `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`.
- `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada no `changeStatus` [09:41 Bruno].
- CRUD de configuração, rotação de secret, histórico de entregas e replay de DLQ (ADMIN) [09:31–09:36].
- Worker `src/worker.ts` + `src/modules/webhooks/webhook.worker.ts` [09:11 Larissa, 09:28 Bruno].

**Fora do escopo** (origem na reunião)

| ID | Exclusão | Origem |
|---|---|---|
| FDD-EXC-01 | Webhooks de entrada (inbound) | [09:02 Marcos, 09:03 Sofia] |
| FDD-EXC-02 | Notificação por e-mail em falhas repetidas (próxima fase) | [09:37 Larissa] |
| FDD-EXC-03 | Rate limiting de saída por cliente (observar e decidir) | [09:39 Diego, 09:39 Larissa] |
| FDD-EXC-04 | Dashboard/painel visual (projeto do time de frontend) | [09:40 Larissa] |
| FDD-EXC-05 | Arquivamento de linhas entregues (~30 dias) | [09:08 Diego] |
| FDD-EXC-06 | Múltiplos workers em paralelo / ordenação global | [09:12 Diego, 09:13 Larissa] |
| FDD-EXC-07 | Itens do pedido no payload | [09:43 Diego] |
| FDD-EXC-08 | Evento na **criação** do pedido. O gancho acordado é só o `changeStatus`, e `OrderService.create` não é alterado | [09:40 Bruno] · `src/modules/orders/order.service.ts` |

## 4. Fluxos detalhados

### 4.1 FDD-FLX-01 — Criação do evento na outbox (dentro do `changeStatus`)

```
PATCH /api/v1/orders/:id/status  (authenticate → validate → OrderController.changeStatus)
└─ OrderService.changeStatus → prisma.$transaction(async (tx) => {
     1. tx.order.findUnique({ id, include: items })          (existente)
     2. valida from≠to e canTransition(from, to)              (existente)
     3. debitStock / replenishStock                            (existente)
     4. tx.order.update({ status: to })                        (existente)
     5. tx.orderStatusHistory.create({...})                    (existente)
     6. await publishWebhookEvent(tx, order, from, to)         ◄── NOVO
        a. webhooks = tx.webhookEndpoint.findMany({
              where: { customerId: order.customerId, active: true } })
        b. alvo = webhooks.filter(w => w.events.includes(to))   (filtro na inserção)
        c. se alvo.length === 0 → return 0   (nenhuma linha gravada)
        d. eventId = randomUUID(); payload = buildPayload(eventId, order, from, to)
        e. tx.webhookOutbox.createMany(alvo.map(w => ({
              eventId, webhookId: w.id, orderId: order.id,
              eventType: 'order.status_changed', payload,
              status: 'PENDING', attempts: 0, nextAttemptAt: now })))
        f. logger.info({ eventId, orderId, webhookIds, toStatus: to }, 'webhook_event_enqueued')
     7. tx.order.findUnique(... include ...)                   (existente) → resposta
   })
```

Regras:

- **R1 (atomicidade):** qualquer exceção em 6 propaga e faz rollback da transação inteira. O status não muda e a API responde como qualquer erro não tratado (500, via `error.middleware.ts`) [09:40 Bruno, 09:41 Diego].
- **R2 (filtro na inserção):** só existe linha para webhooks **ativos** do `customer_id` do pedido que assinam o `to_status` [09:33 Marcos, 09:34 Bruno].
- **R3 (fan-out):** uma linha por webhook-alvo, todas com o **mesmo `eventId`** ("único por evento" [09:25 Diego]). O `id` da linha é outro UUID. Assim cada endpoint tem ciclo de retry e DLQ próprio, porque cada um tem URL e secret distintas [09:21 Sofia]. **(escolha de implementação)**
- **R4 (snapshot):** o `payload` é montado aqui, com os valores do momento da mudança, e nunca recalculado [09:52 Larissa] ([ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md)).
- **R5:** o `order` usado em `buildPayload` é o objeto lido no passo 1. `from` e `to` vêm como parâmetros. `total_cents` e `order_number` não mudam no `changeStatus`.

### 4.2 FDD-FLX-02 — Processamento pelo worker

```
src/worker.ts (bootstrap)
 ├─ prisma = createPrismaClient()                      (instância própria [09:30 Bruno])
 ├─ recoverStuck(): UPDATE webhook_outbox SET status='PENDING' WHERE status='PROCESSING'
 └─ loop:  runTick(); setTimeout(loop, WEBHOOK_POLL_INTERVAL_MS=2000)   (sem sobreposição de ticks)

runTick()  — src/modules/webhooks/webhook.worker.ts
 1. claim (transação curta):
      rows = SELECT … FROM webhook_outbox
             WHERE status='PENDING' AND next_attempt_at <= NOW()
             ORDER BY created_at ASC LIMIT WEBHOOK_BATCH_SIZE
      UPDATE webhook_outbox SET status='PROCESSING' WHERE id IN (rows)
 2. para cada row, EM SEQUÊNCIA (preserva ordem de created_at [09:12 Diego]):
      deliver(row)

deliver(row)
 a. webhook = findUnique(row.webhookId)
      ├─ inexistente/inativo → deadLetter(row, 'WEBHOOK_INACTIVE')            (sem retry)
      └─ sem secret            → deadLetter(row, 'WEBHOOK_SECRET_REQUIRED')   (sem retry)
 b. body = JSON.stringify(row.payload)
      └─ Buffer.byteLength(body) > 65 536 → deadLetter(row, 'WEBHOOK_PAYLOAD_TOO_LARGE') (sem retry)
 c. headers = buildHeaders(row, webhook, body)            (ver §6.3)
 d. t0 = now; res = fetch(webhook.url, { method:'POST', headers, body,
                                         signal: AbortSignal.timeout(10_000) })
 e. INSERT webhook_deliveries (attempt = row.attempts+1, statusCode, responseBody truncado,
                               durationMs, success, errorCode)
 f. 2xx      → UPDATE status='DELIVERED', delivered_at=NOW(), attempts+1
    senão    → handleFailure(row, errorCode)               (FDD-FLX-03)
```

Regras:

- **R6:** o worker é **single-instance** nesta fase. O `recoverStuck()` na partida só é seguro por isso [09:12 Diego].
- **R7:** a checagem de 64 KB acontece **no worker, não na transação**. Um erro dentro do `changeStatus` desfaria a mudança de status, contrariando [09:40 Bruno]. No worker o evento "não é enviado e vira erro", como decidido [09:23 Sofia, 09:24 Larissa], mas o status do pedido segue intacto. **(escolha de implementação)**
- **R8:** webhook desativado ou removido depois do enfileiramento: o evento não é enviado e vai direto para a DLQ com `WEBHOOK_INACTIVE`, reprocessável após reativação. **(escolha de implementação)**

### 4.3 FDD-FLX-03 — Retry com backoff

```
BACKOFF = [1m, 5m, 30m, 2h, 12h]          [09:17 Larissa]
MAX_ATTEMPTS = 1 + BACKOFF.length = 6     (1 envio inicial + 5 retentativas — ver RFC Q-06)

handleFailure(row, errorCode):
  attempts = row.attempts + 1
  if attempts < MAX_ATTEMPTS:
      UPDATE status='PENDING', attempts, next_attempt_at = NOW() + BACKOFF[attempts-1],
             last_error = errorCode
      logger.warn({...}, 'webhook_delivery_retry_scheduled')
  else:
      deadLetter(row, 'WEBHOOK_MAX_ATTEMPTS_EXCEEDED')
```

| Tentativa | Momento (a partir da 1ª falha) | Intervalo |
|---|---|---|
| 1 (envio inicial) | t₀ (≤ 2s após commit) | — |
| 2 | t₀ + 1 min | 1 min |
| 3 | t₀ + 6 min | 5 min |
| 4 | t₀ + 36 min | 30 min |
| 5 | t₀ + 2h36 | 2 h |
| 6 (última) | t₀ + 14h36 ("quase 15 horas" [09:17 Diego]) | 12 h |

Contam como falha:

- timeout de 10s (`WEBHOOK_DELIVERY_TIMEOUT`) [09:42 Diego];
- erro de rede, DNS ou TLS (`WEBHOOK_DELIVERY_NETWORK_ERROR`);
- qualquer resposta não-2xx (`WEBHOOK_DELIVERY_HTTP_ERROR`). A reunião não distinguiu 4xx de 5xx, então ambos entram em retry.

### 4.4 FDD-FLX-04 — DLQ e replay manual

```
deadLetter(row, errorCode):   (uma transação)
  UPDATE webhook_outbox SET status='FAILED', last_error=errorCode
  INSERT webhook_dead_letter (outbox_id, event_id, webhook_id, payload, error_code,
                              failure_reason, attempts, failed_at=NOW())
  logger.error({...}, 'webhook_dead_lettered')

POST /api/v1/admin/webhooks/dead-letter/:id/replay   (authenticate → requireRole('ADMIN'))
  1. dl = findUnique(:id)                   → 404 WEBHOOK_DEAD_LETTER_NOT_FOUND
  2. dl.replayedAt != null                  → 409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED
  3. outbox/webhook ausente                 → 404 WEBHOOK_NOT_FOUND
     webhook inativo                        → 409 WEBHOOK_INACTIVE
  4. transação:
       UPDATE webhook_outbox SET status='PENDING', attempts=0, next_attempt_at=NOW(), last_error=NULL
       UPDATE webhook_dead_letter SET replayed_at=NOW(), replayed_by_id=req.user.id
  5. logger.info({ deadLetterId, outboxId, eventId, replayedBy: req.user.id }, 'webhook_dead_letter_replayed')
  6. 202
```

- O replay **reusa a mesma linha da outbox** e, portanto, o mesmo `event_id`. É o que permite ao cliente deduplicar se já tiver recebido o evento ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)) [09:18 Diego].
- Quem fez o replay fica registrado em log **e** em `replayed_by_id` [09:36 Sofia].
- Se falhar de novo após o replay, uma **nova** linha de DLQ é criada, e a anterior mantém o histórico do replay.

### 4.5 FDD-FLX-05 — Rotação de secret

```
POST /api/v1/webhooks/:id/rotate-secret
  previousSecret = current.secret
  previousSecretExpiresAt = NOW() + 24h         [09:21 Sofia]
  secret = randomBytes(32).toString('hex')
  → 200 { secret (nova, exibida só aqui), previousSecretExpiresAt }
Worker, ao assinar: se previousSecret && previousSecretExpiresAt > NOW() → assina com as duas (§6.3)
```

- No máximo **duas** secrets válidas: a atual e a imediatamente anterior. Uma nova rotação durante o grace descarta a mais antiga. **(escolha de implementação)**
- A limpeza é preguiçosa: `previousSecret` expirado é simplesmente ignorado e anulado na próxima escrita do registro.

### 4.6 FDD-FLX-06 — Shutdown e recuperação

- `src/worker.ts` trata `SIGINT` e `SIGTERM` como `src/server.ts`: para de agendar ticks, espera o `deliver` em curso terminar, faz `prisma.$disconnect()` e sai.
- Se o processo morrer no meio (`PROCESSING` órfão), o `recoverStuck()` da próxima partida devolve a linha para `PENDING`. Um reenvio duplicado é aceitável pela garantia at-least-once [09:24 Diego].

## 5. Modelo de dados

Adições em `prisma/schema.prisma`, com migration via `npm run db:migrate`. Todas as PKs são UUID `Char(36)`, no padrão do projeto [09:51 Larissa].

```prisma
enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  DELIVERED
  FAILED
}

model WebhookEndpoint {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(128)
  previousSecret          String?   @db.VarChar(128)
  previousSecretExpiresAt DateTime?
  events                  Json      // OrderStatus[] assinados
  active                  Boolean   @default(true)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer   Customer            @relation(fields: [customerId], references: [id])
  outbox     WebhookOutbox[]
  deliveries WebhookDelivery[]

  @@index([customerId, active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36)
  eventId       String              @db.Char(36)
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)   // sem FK: snapshot sobrevive a OrderService.delete
  eventType     String              @db.VarChar(64)
  payload       Json
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastError     String?             @db.VarChar(500)
  deliveredAt   DateTime?
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  webhook    WebhookEndpoint   @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  deliveries WebhookDelivery[]

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
  attemptNumber  Int
  success        Boolean
  responseStatus Int?
  responseBody   String?  @db.Text        // truncado
  durationMs     Int
  errorCode      String?  @db.VarChar(64)
  createdAt      DateTime @default(now())

  outbox  WebhookOutbox   @relation(fields: [outboxId], references: [id], onDelete: Cascade)
  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id            String    @id @default(uuid()) @db.Char(36)
  outboxId      String    @db.Char(36)
  eventId       String    @db.Char(36)
  webhookId     String    @db.Char(36)
  payload       Json
  errorCode     String    @db.VarChar(64)
  failureReason String    @db.VarChar(500)
  attempts      Int
  failedAt      DateTime  @default(now())
  replayedAt    DateTime?
  replayedById  String?   @db.Char(36)

  @@index([failedAt])
  @@map("webhook_dead_letter")
}
```

Notas:

- **Índices:** `status` + `next_attempt_at` e `created_at` atendem a consulta do worker [09:08 Diego]. `next_attempt_at` foi acrescentado para suportar o backoff.
- **`Customer`** ganha só o campo de relação `webhooks WebhookEndpoint[]`, exigido pelo Prisma. Nenhuma coluna existente muda.
- **`webhook_dead_letter`** não tem FK de propósito: é evidência e precisa sobreviver à remoção do webhook [09:18 Diego].

## 6. Contratos públicos

Todas as rotas ficam sob `/api/v1` (`src/app.ts`), exigem `Authorization: Bearer <JWT>` (`authenticate`) e retornam erros no envelope `{ "error": { "code", "message", "details?" } }` do `error.middleware.ts`.

O `customerId` vai no **body** (criação) ou na **query** (listagem). Ele **não** vem do JWT, que é do usuário operador [09:32 Bruno, 09:32 Larissa].

### 6.1 API de configuração

#### FDD-CONTRATO-01 — `POST /api/v1/webhooks` (cadastrar)

[09:31 Marcos] · qualquer role autenticada [09:36 Marcos, 09:37 Sofia]

```http
POST /api/v1/webhooks
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{
  "customerId": "7b0c1d3e-2f4a-4b5c-8d9e-0f1a2b3c4d5e",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

```http
HTTP/1.1 201 Created
{
  "id": "c2a9f1e0-5b6d-4e7f-8a9b-0c1d2e3f4a5b",
  "customerId": "7b0c1d3e-2f4a-4b5c-8d9e-0f1a2b3c4d5e",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
  "createdAt": "2026-10-01T12:00:00.000Z",
  "updatedAt": "2026-10-01T12:00:00.000Z"
}
```

| Status | Quando |
|---|---|
| 201 | Criado. **A `secret` só é devolvida aqui e na rotação** [09:31 Marcos] |
| 400 | `VALIDATION_ERROR`: URL não `https` (detail `WEBHOOK_INVALID_URL`), `events` vazio ou com status inválido |
| 401 | `UNAUTHORIZED` |
| 404 | `WEBHOOK_CUSTOMER_NOT_FOUND` |

Schema Zod (`webhook.schemas.ts`):

- `url`: `z.string().url().max(2048)` com `refine(u => u.startsWith('https://'), 'WEBHOOK_INVALID_URL: url must use https')` [09:23 Sofia];
- `events`: `z.array(z.nativeEnum(OrderStatus)).min(1)`, **excluindo `PENDING`**, porque nenhuma transição em `order.status.ts` leva a `PENDING` e o evento nunca aconteceria.

#### FDD-CONTRATO-02 — `GET /api/v1/webhooks?customerId=…&page=1&pageSize=20` (listar)

[09:33 Bruno]

```http
HTTP/1.1 200 OK
{
  "data": [
    {
      "id": "c2a9f1e0-5b6d-4e7f-8a9b-0c1d2e3f4a5b",
      "customerId": "7b0c1d3e-2f4a-4b5c-8d9e-0f1a2b3c4d5e",
      "url": "https://hooks.atlascomercial.com.br/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "previousSecretExpiresAt": null,
      "createdAt": "2026-10-01T12:00:00.000Z",
      "updatedAt": "2026-10-01T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

- `customerId` é obrigatório na query.
- A resposta usa `paginated()` de `src/shared/http/response.ts`.
- **Nunca devolve `secret`.**
- Status: `200`, `400 VALIDATION_ERROR`, `401`.

#### FDD-CONTRATO-03 — `PATCH /api/v1/webhooks/:id` (editar)

[09:33 Bruno]

```http
PATCH /api/v1/webhooks/c2a9f1e0-5b6d-4e7f-8a9b-0c1d2e3f4a5b
{ "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"], "active": true }
```

```http
HTTP/1.1 200 OK
{ "id": "c2a9f1e0-...", "customerId": "7b0c1d3e-...", "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"], "active": true,
  "previousSecretExpiresAt": null, "createdAt": "...", "updatedAt": "2026-10-02T09:30:00.000Z" }
```

- Campos aceitos: `url`, `events`, `active`, todos opcionais, com pelo menos um. As mesmas regras Zod do POST valem aqui.
- `customerId` e `secret` não são editáveis.
- Status: `200`, `400 VALIDATION_ERROR`, `401`, `404 WEBHOOK_NOT_FOUND`.

#### FDD-CONTRATO-04 — `DELETE /api/v1/webhooks/:id` (remover)

[09:33 Bruno]

```http
DELETE /api/v1/webhooks/c2a9f1e0-5b6d-4e7f-8a9b-0c1d2e3f4a5b
→ HTTP/1.1 204 No Content
```

- Remove o endpoint. Linhas de `webhook_outbox` e `webhook_deliveries` saem em cascata. `webhook_dead_letter` é preservada.
- Status: `204`, `401`, `404 WEBHOOK_NOT_FOUND`.

#### FDD-CONTRATO-05 — `POST /api/v1/webhooks/:id/rotate-secret` (rotacionar secret)

[09:21 Sofia]

```http
POST /api/v1/webhooks/c2a9f1e0-5b6d-4e7f-8a9b-0c1d2e3f4a5b/rotate-secret
(sem body)
```

```http
HTTP/1.1 200 OK
{
  "id": "c2a9f1e0-5b6d-4e7f-8a9b-0c1d2e3f4a5b",
  "secret": "3c9909afec25354d551dae21590bb26e38d53f2173b8d3dc3eee4c047e7ab1c1",
  "previousSecretExpiresAt": "2026-10-03T14:00:00.000Z"
}
```

- Status: `200`, `401`, `404 WEBHOOK_NOT_FOUND`, `409 WEBHOOK_INACTIVE`. Não se rotaciona secret de endpoint desativado. **(escolha de implementação)**

#### FDD-CONTRATO-06 — `GET /api/v1/webhooks/:id/deliveries` (histórico de entregas)

[09:34 Marcos]

```http
GET /api/v1/webhooks/c2a9f1e0-5b6d-4e7f-8a9b-0c1d2e3f4a5b/deliveries?limit=100
```

```http
HTTP/1.1 200 OK
{
  "data": [
    {
      "id": "e1f2a3b4-...",
      "eventId": "0e6f3a52-8d1b-4c7e-9f20-5a4b3c2d1e0f",
      "attemptNumber": 2,
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "durationMs": 184,
      "errorCode": null,
      "createdAt": "2026-10-01T12:01:03.412Z",
      "payload": { "event_id": "0e6f3a52-...", "event_type": "order.status_changed", "...": "..." }
    },
    {
      "id": "d0e1f2a3-...",
      "eventId": "0e6f3a52-8d1b-4c7e-9f20-5a4b3c2d1e0f",
      "attemptNumber": 1,
      "success": false,
      "responseStatus": null,
      "responseBody": null,
      "durationMs": 10000,
      "errorCode": "WEBHOOK_DELIVERY_TIMEOUT",
      "createdAt": "2026-10-01T12:00:02.020Z",
      "payload": { "...": "..." }
    }
  ]
}
```

- Retorna as **últimas 100 tentativas** em ordem decrescente de `createdAt`: `limit` de 1 a 100, padrão 100.
- Cada item traz sucesso/falha, payload, resposta e tempo de resposta [09:34 Marcos].
- Status: `200`, `400`, `401`, `404 WEBHOOK_NOT_FOUND`.

### 6.2 API administrativa

#### FDD-CONTRATO-07 — `POST /api/v1/admin/webhooks/dead-letter/:id/replay`

[09:18 Diego] · **`requireRole('ADMIN')`** [09:36 Sofia, 09:36 Larissa]

```http
POST /api/v1/admin/webhooks/dead-letter/5d4c3b2a-1f0e-4d9c-8b7a-6f5e4d3c2b1a/replay
Authorization: Bearer <JWT de ADMIN>
```

```http
HTTP/1.1 202 Accepted
{
  "deadLetterId": "5d4c3b2a-1f0e-4d9c-8b7a-6f5e4d3c2b1a",
  "outboxId": "a1b2c3d4-...",
  "eventId": "0e6f3a52-8d1b-4c7e-9f20-5a4b3c2d1e0f",
  "status": "PENDING",
  "replayedAt": "2026-10-02T10:00:00.000Z",
  "replayedById": "9a8b7c6d-5e4f-4a3b-8c2d-1e0f9a8b7c6d"
}
```

| Status | Quando |
|---|---|
| 202 | Evento recolocado na outbox como `PENDING` |
| 401 / 403 | `UNAUTHORIZED` / `FORBIDDEN` (role OPERATOR) |
| 404 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` ou `WEBHOOK_NOT_FOUND` (endpoint removido) |
| 409 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` ou `WEBHOOK_INACTIVE` |

### 6.3 FDD-CONTRATO-08 — Contrato de saída (OMS → cliente)

```http
POST https://hooks.atlascomercial.com.br/oms
Content-Type: application/json
X-Event-Id: 0e6f3a52-8d1b-4c7e-9f20-5a4b3c2d1e0f
X-Webhook-Id: c2a9f1e0-5b6d-4e7f-8a9b-0c1d2e3f4a5b
X-Timestamp: 2026-10-01T12:00:02.004Z
X-Signature: sha256=5257a869e7ecebeda32affa62cdca3fa51cad7e77a0e56ff536d0ce8e108d8bd

{
  "event_id": "0e6f3a52-8d1b-4c7e-9f20-5a4b3c2d1e0f",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-01T12:00:00.518Z",
  "order_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "order_number": "ORD-000123",
  "customer_id": "7b0c1d3e-2f4a-4b5c-8d9e-0f1a2b3c4d5e",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "total_cents": 18500
}
```

| Header | Conteúdo | Origem |
|---|---|---|
| `Content-Type` | `application/json` | [09:44 Diego] |
| `X-Event-Id` | UUID do evento, estável em retentativas e replay | [09:25 Diego] |
| `X-Webhook-Id` | `id` do cadastro do endpoint | [09:44 Sofia, 09:45 Diego] |
| `X-Timestamp` | ISO 8601 do **envio** (a cada tentativa) | [09:44 Diego] |
| `X-Signature` | `sha256=` + hex de `HMAC-SHA256(secret, rawBody)`. No grace de rotação, duas assinaturas separadas por vírgula (proposta, RFC Q-07) | [09:20 Sofia, 09:22 Sofia] |

- **Campos do corpo:** `timestamp` é o instante da mudança de status (snapshot). `order_number` segue o formato `ORD-000123`, gerado por `reserveOrderNumber` em `order.service.ts`. Não há `items` [09:43 Diego].
- **Resposta esperada do cliente:** qualquer `2xx` em até **10s** [09:42 Diego]. O corpo da resposta é gravado truncado em `webhook_deliveries`.
- **Verificação no cliente (portal):** recalcular o HMAC sobre o corpo bruto, comparar em tempo constante e deduplicar por `X-Event-Id` [09:26 Marcos].

## 7. Matriz de erros

Todos os erros de API são subclasses de `AppError` com prefixo `WEBHOOK_` [09:28 Bruno, 09:29 Larissa]. Os erros internos do worker usam os mesmos códigos em `last_error`, `webhook_deliveries.error_code`, `webhook_dead_letter.error_code` e nos logs.

| Código | HTTP | Classe (base) | Origem | Quando | Retry? |
|---|---|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | `WebhookNotFoundError` (`AppError`) | API | `:id` inexistente em PATCH/DELETE/rotate/deliveries, ou endpoint removido no replay | — |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `WebhookCustomerNotFoundError` (`AppError`) | API | `customerId` inexistente no POST | — |
| `WEBHOOK_INVALID_URL` | 400 | Zod → `ValidationError` (`VALIDATION_ERROR`, code no `details[].message`) | API | URL não-`https` ou malformada [09:23 Sofia] | — |
| `WEBHOOK_INACTIVE` | 409 / interno | `WebhookInactiveError` (`ConflictError`) | API + worker | Rotate ou replay em endpoint inativo. No worker: envio para endpoint desativado → DLQ | Não |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `WebhookDeadLetterNotFoundError` (`AppError`) | API | `:id` de DLQ inexistente | — |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | `WebhookDeadLetterAlreadyReplayedError` (`ConflictError`) | API | Replay repetido da mesma entrada | — |
| `WEBHOOK_SECRET_REQUIRED` | interno | — | worker | Endpoint sem secret: não há como assinar [09:28 Bruno] | Não → DLQ |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | interno | — | worker | Corpo serializado > 64 KB [09:24 Larissa] | Não → DLQ |
| `WEBHOOK_DELIVERY_TIMEOUT` | interno | — | worker | Sem resposta em 10s [09:42 Diego] | Sim |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | interno | — | worker | Resposta não-2xx | Sim |
| `WEBHOOK_DELIVERY_NETWORK_ERROR` | interno | — | worker | DNS, conexão recusada, TLS | Sim |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | interno | — | worker | Falhou a 6ª tentativa [09:17 Larissa] | → DLQ |
| `UNAUTHORIZED` / `FORBIDDEN` | 401 / 403 | existentes (`http-errors.ts`) | API | Sem JWT / não-ADMIN no replay [09:36 Sofia] | — |

> Por que `WEBHOOK_INVALID_URL` sai como `VALIDATION_ERROR`: `validate.middleware.ts` converte todo `ZodError` em `ValidationError` com o código fixo `VALIDATION_ERROR`. A regra fica no Zod, como Sofia pediu [09:23 Sofia], e o código `WEBHOOK_INVALID_URL` segue no `details[].message` sem mudar o middleware. A alternativa é lançar `WebhookInvalidUrlError` no service, o que duplicaria a validação. A escolha está registrada no Tracker.

## 8. Estratégias de resiliência

| ID | Mecanismo | Valor | Origem |
|---|---|---|---|
| FDD-RES-01 | Desacoplamento via outbox transacional | insert na mesma `$transaction` | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) [09:06 Diego] |
| FDD-RES-02 | Timeout por chamada HTTP | 10s (`AbortSignal.timeout`) | [09:42 Diego] |
| FDD-RES-03 | Retry com backoff exponencial fixo | 1m, 5m, 30m, 2h, 12h | [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md) [09:17 Larissa] |
| FDD-RES-04 | Teto de tentativas | 1 + 5 (≈ 14,6h de janela) | [09:15 Diego, 09:17 Diego] |
| FDD-RES-05 | Fallback: DLQ + replay manual ADMIN | tabela `webhook_dead_letter` | [09:18 Diego] |
| FDD-RES-06 | Falhas não recuperáveis vão direto para DLQ | payload > 64 KB, sem secret, inativo | [09:24 Larissa] + §4.2 |
| FDD-RES-07 | Polling sem sobreposição | `setTimeout` após o fim do tick, intervalo 2s | [09:09 Diego] |
| FDD-RES-08 | Lote pequeno por tick | `WEBHOOK_BATCH_SIZE` = 10 **(escolha de implementação)** | "batch pequeno" [09:08 Diego] |
| FDD-RES-09 | Recuperação de `PROCESSING` órfão na partida | `recoverStuck()` | at-least-once [09:24 Diego] |
| FDD-RES-10 | Isolamento de processo | worker ≠ API | [09:11 Diego] |
| FDD-RES-11 | Fallback de notificação por e-mail | **não nesta fase** | [09:37 Larissa] |

Configuração em `src/config/env.ts`, com defaults e sem obrigar mudança no `.env`:

- `WEBHOOK_POLL_INTERVAL_MS` = 2000
- `WEBHOOK_HTTP_TIMEOUT_MS` = 10000
- `WEBHOOK_BATCH_SIZE` = 10

## 9. Observabilidade

Não há biblioteca de métricas nem de tracing no projeto, e a decisão é não adicionar dependências [09:29 Bruno]. A observabilidade se apoia no **Pino** (logs estruturados JSON) e em **consultas SQL** às tabelas do módulo.

### 9.1 Logs (Pino — `src/shared/logger/index.ts`)

O worker usa `logger.child({ component: 'webhook-worker' })`. Eventos, todos com `eventId`, `webhookId`, `outboxId` e `orderId` quando aplicáveis:

| Mensagem | Nível | Campos adicionais |
|---|---|---|
| `webhook_event_enqueued` | info | `toStatus`, `fromStatus`, `webhookIds[]` |
| `webhook_delivery_attempt` | info / warn | `attempt`, `statusCode`, `durationMs`, `success`, `errorCode` |
| `webhook_delivery_retry_scheduled` | warn | `attempt`, `nextAttemptAt` |
| `webhook_dead_lettered` | error | `errorCode`, `attempts` |
| `webhook_dead_letter_replayed` | info | `deadLetterId`, `replayedBy` (userId) [09:36 Sofia] |
| `webhook_secret_rotated` | info | `userId`, `previousSecretExpiresAt` (**sem a secret**) |
| `webhook_worker_started` / `webhook_worker_tick` / `webhook_worker_shutdown` | info / debug | `batchSize`, `claimed`, `durationMs` |

**Redaction:** acrescentar `'*.secret'` e `'*.previousSecret'` a `redactPaths` em `src/shared/logger/index.ts` [09:22 Diego].

### 9.2 Métricas

Derivadas de logs agregados e de gauges via SQL:

| Métrica | Tipo | Fonte | Uso |
|---|---|---|---|
| `webhook_event_delivery_latency_ms` (`delivered_at − created_at`, 1ª tentativa) | histograma | `webhook_outbox` | Meta p95 < 10s [09:02 Marcos] |
| `webhook_delivery_attempts_total{success}` | contador | log `webhook_delivery_attempt` | Taxa de sucesso por webhook |
| `webhook_delivery_duration_ms` | histograma | `webhook_deliveries.duration_ms` | Clientes lentos (perto de 10s) |
| `webhook_outbox_due_total` | gauge | `COUNT(*) WHERE status='PENDING' AND next_attempt_at<=NOW()` | Backlog |
| `webhook_outbox_oldest_due_age_seconds` | gauge | `NOW() − MIN(created_at)` dos devidos | **Alarme de worker parado** (> 10s sustentado) |
| `webhook_dead_letter_total{error_code}` | contador | `webhook_dead_letter` | Falhas permanentes |
| `webhook_events_per_webhook_per_minute` | gauge | `webhook_outbox` | Insumo para decidir rate limiting (RFC Q-01) [09:39 Diego] |

### 9.3 Tracing (correlação)

- **`event_id`** é o identificador de trace ponta a ponta: é gerado na transação da API, aparece em todos os logs do worker e em `webhook_deliveries`, e é enviado ao cliente em `X-Event-Id`. Os logs dos dois lados se cruzam por ele [09:25 Diego].
- **`order_id`** liga o evento ao log `http_request` do `PATCH /orders/:id/status` (`request-logger.middleware.ts`, que já emite `requestId` e `path`).
- **`X-Webhook-Id`** permite ao cliente com vários cadastros saber qual deles gerou o envio [09:44 Sofia].
- Tracing distribuído (OpenTelemetry) **não** faz parte desta fase porque o projeto não tem essa infraestrutura.

## 10. Integração com o sistema existente

| # | Arquivo (existente) | Como o módulo de webhooks se integra |
|---|---|---|
| I-01 | `src/modules/orders/order.service.ts` | **Alteração crítica.** Em `changeStatus`, após `tx.orderStatusHistory.create(...)` e antes do `findUnique` final, chamar `await publishWebhookEvent(tx, order, from, to)`. O `tx` é o `Prisma.TransactionClient` já tipado como `TxClient` no arquivo. Uma falha propaga e faz rollback [09:40 Bruno, 09:41 Bruno]. O construtor do `OrderService` **não muda**: a função é importada, não injetada [09:41 Diego]. |
| I-02 | `src/modules/orders/order.status.ts` | Sem alteração. É a referência para validar `events`: como nenhuma transição leva a `PENDING`, o schema rejeita `PENDING` no filtro. |
| I-03 | `prisma/schema.prisma` | Novos `enum WebhookOutboxStatus` e models `WebhookEndpoint`, `WebhookOutbox`, `WebhookDelivery`, `WebhookDeadLetter` (§5). `Customer` ganha só a relação `webhooks`. Migration nova em `prisma/migrations/`. |
| I-04 | `src/shared/errors/http-errors.ts` e `src/shared/errors/index.ts` | Novas classes `Webhook*Error` estendendo `AppError` / `ConflictError`, no mesmo molde de `InvalidStatusTransitionError` e `InsufficientStockError`, exportadas pelo `index.ts` [09:28 Bruno]. `WebhookNotFoundError` estende `AppError` diretamente, porque `NotFoundError` fixa `NOT_FOUND`. |
| I-05 | `src/shared/errors/app-error.ts` | Sem alteração. É a base (`statusCode`, `errorCode`, `details`) de todos os erros do módulo. |
| I-06 | `src/middlewares/error.middleware.ts` | **Sem alteração.** Já serializa `AppError`, `ZodError` e Prisma `P2002`/`P2025` [09:29 Bruno]. |
| I-07 | `src/middlewares/auth.middleware.ts` | `authenticate` em todas as rotas de webhook. `requireRole('ADMIN')` no replay [09:36 Larissa]. `req.user.id` alimenta `replayed_by_id` e os logs de auditoria. |
| I-08 | `src/middlewares/validate.middleware.ts` | `validate({ body, params, query })` com os schemas de `webhook.schemas.ts`, como em `order.routes.ts`. |
| I-09 | `src/shared/logger/index.ts` | Reuso do `logger` e `logger.child` no worker. Acrescentar `*.secret` e `*.previousSecret` a `redactPaths`. |
| I-10 | `src/config/database.ts` | O worker chama `createPrismaClient()` para obter **seu próprio** `PrismaClient`, com a mesma `DATABASE_URL` [09:30 Bruno]. |
| I-11 | `src/config/env.ts` | Três variáveis novas com default no `envSchema` Zod (§8). |
| I-12 | `src/server.ts` | Modelo para a nova entry-point `src/worker.ts`: `bootstrap()`, handlers de `SIGINT`/`SIGTERM` e `prisma.$disconnect()` [09:11 Larissa]. |
| I-13 | `src/app.ts` | Em `buildControllers()`: instanciar `WebhookRepository` → `WebhookService` → `WebhookController` e expor como `webhooks`. |
| I-14 | `src/routes/index.ts` | Adicionar `webhooks: WebhookController` ao tipo `Controllers`, `router.use('/webhooks', buildWebhookRouter(...))` e `router.use('/admin/webhooks', buildWebhookAdminRouter(...))`. |
| I-15 | `src/shared/http/response.ts` | `paginated()` na listagem de webhooks. |
| I-16 | `package.json` | Scripts `"worker": "node --env-file=.env dist/worker.js"` e `"worker:dev": "tsx watch --env-file=.env src/worker.ts"`, espelhando `start`/`dev` [09:11 Larissa]. `tsconfig.build.json` já compila `src/`. |
| I-17 | `tests/setup.ts` e `tests/helpers/factories.ts` | Limpar as tabelas `webhook_*` no `beforeEach` antes de `order`/`customer`. Nova factory `createTestWebhook()`. |

**Arquivos novos** em `src/modules/webhooks/` [09:27 Bruno, 09:28 Bruno]:

- `webhook.routes.ts`, `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.schemas.ts`;
- `webhook.publisher.ts` (`publishWebhookEvent`, `buildPayload`);
- `webhook.signer.ts` (HMAC);
- `webhook.worker.ts` (`runTick`, `deliver`, `handleFailure`, `deadLetter`);
- mais `src/worker.ts`.

## 11. Dependências e compatibilidade

- **Runtime:**
  - Node ≥ 20 (`engines` em `package.json`), que já traz `fetch`, `AbortSignal.timeout`, `crypto.randomUUID`, `crypto.createHmac` e `crypto.randomBytes`;
  - `uuid` 11 já está disponível;
  - **nenhuma dependência npm nova**.
- **Banco:** MySQL 8.0 (`docker-compose.yml`) com colunas `JSON`. Prisma 5.22 (`createMany` suportado em MySQL).
- **Compatibilidade de API:** nenhum contrato existente muda. `PATCH /orders/:id/status` mantém request e response. Sem webhooks cadastrados, o comportamento é idêntico ao atual, porque o `findMany` retorna vazio e nada é gravado.
- **Latência do `changeStatus`:** ganha um `SELECT` indexado (`customer_id, active`) e, havendo assinantes, um `createMany`.
- **Deploy:**
  1. migration;
  2. API;
  3. worker.
  Se a API for implantada antes do worker, os eventos acumulam na outbox e são entregues quando o worker subir.
- **Dependências externas:**
  - endpoints HTTPS dos clientes;
  - portal de desenvolvedor mantido por Marcos [09:26 Marcos, 09:40 Marcos];
  - revisão de segurança da Sofia (≥ 2 dias úteis) antes do deploy [09:46 Sofia].

## 12. Riscos e mitigação

| ID | Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|---|
| FDD-RISK-01 | Worker parado ou travado → backlog cresce, latência estoura 10s | Média | Alto | Alarme em `webhook_outbox_oldest_due_age_seconds`. `recoverStuck()`. Deploy separado [09:11 Diego] |
| FDD-RISK-02 | Um cliente lento (10s × lote) atrasa os outros, porque o processamento é sequencial | Média | Médio | Lote pequeno. Monitorar `webhook_delivery_duration_ms`. Paralelizar entre webhooks distintos fica para a evolução de escala (RFC Q-03) |
| FDD-RISK-03 | Entrega fora de ordem para o mesmo pedido quando há retry | Média | Baixo | At-least-once sem garantia de ordem, documentado. O payload tem `from_status`/`to_status` e `timestamp`. RFC Q-08 [09:13 Larissa, 09:14 Marcos] |
| FDD-RISK-04 | Secret vazada em log (nosso ou do cliente) | Baixa | Alto | Redaction no Pino. Secret exibida só em create/rotate. Rotação com grace de 24h [09:22 Diego] |
| FDD-RISK-05 | Falha no insert da outbox bloqueia mudanças de status | Baixa | Alto | Comportamento intencional (consistência). Testes de integração cobrem o rollback [09:40 Bruno] |
| FDD-RISK-06 | Crescimento contínuo de `webhook_outbox`/`webhook_deliveries` | Alta | Baixo | Índices adequados. Arquivamento explicitamente fora do escopo, registrado como dívida [09:08 Diego] |
| FDD-RISK-07 | Cliente não deduplica e processa o evento duas vezes | Média | Médio | `X-Event-Id` + documentação no portal [09:26 Marcos] |
| FDD-RISK-08 | Implementação fraca de HMAC ou geração de secret | Baixa | Alto | `node:crypto` (`randomBytes(32)`, `createHmac('sha256')`). Revisão de segurança dedicada [09:46 Sofia] |

## 13. Critérios de aceite técnicos

| ID | Critério | Como verificar |
|---|---|---|
| FDD-CA-01 | Ao mudar o status de um pedido cujo cliente tem webhook ativo assinando o `to_status`, **exatamente uma** linha `PENDING` por webhook-alvo é criada na mesma transação | Teste de integração em `tests/webhooks.test.ts` |
| FDD-CA-02 | Se `publishWebhookEvent` lançar exceção, o status do pedido, o histórico e o estoque **não mudam** | Teste com falha forçada no insert |
| FDD-CA-03 | Sem webhook assinante do `to_status`, **nenhuma** linha é criada [09:34 Bruno] | Teste |
| FDD-CA-04 | O worker entrega em ≤ 2s + latência do cliente. O cliente recebe os headers `Content-Type`, `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp`, `X-Signature` | Teste com servidor HTTP local (`node:http`) |
| FDD-CA-05 | `X-Signature` = `sha256=` + HMAC-SHA256(secret, corpo bruto), verificável pelo receptor | Teste unitário do signer |
| FDD-CA-06 | Timeout de 10s, não-2xx e erro de rede agendam retry em 1m/5m/30m/2h/12h. Após a 6ª falha, a linha fica `FAILED` e é criada uma entrada em `webhook_dead_letter` | Teste com relógio controlado (`vi.useFakeTimers`) ou `nextAttemptAt` manipulado |
| FDD-CA-07 | Payload > 64 KB vai direto para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem chamada HTTP | Teste |
| FDD-CA-08 | O replay exige ADMIN (OPERATOR → 403), volta a linha para `PENDING` com o mesmo `event_id` e registra `replayed_by_id` e o log | Teste |
| FDD-CA-09 | `POST /webhooks` com URL `http://` → 400. A `secret` aparece só em create e rotate, nunca em list/patch | Teste |
| FDD-CA-10 | Após a rotação, envios nas próximas 24h trazem as assinaturas nova e antiga. Depois disso, só a nova | Teste com `previousSecretExpiresAt` manipulado |
| FDD-CA-11 | `GET /webhooks/:id/deliveries` retorna no máximo 100 itens, em ordem decrescente | Teste |
| FDD-CA-12 | Todos os códigos de erro do módulo começam com `WEBHOOK_` e são serializados pelo `error.middleware.ts` sem alteração nele | Revisão + teste |
| FDD-CA-13 | Nenhum log contém o valor de `secret` | Teste do redact / revisão de segurança [09:46 Sofia] |
| FDD-CA-14 | `npm run worker` sobe um processo independente, e matar a API não interrompe entregas | Teste manual / smoke |

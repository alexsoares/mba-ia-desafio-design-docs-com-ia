# RFC — Sistema de Webhooks de Notificação de Pedidos

## 1. Metadados

| Campo | Valor |
|---|---|
| **Autora** | Larissa (Tech Lead), a partir da reunião técnica de kickoff |
| **Status** | Em revisão |
| **Data** | Reunião de kickoff (quinta-feira, 09:00). Revisão marcada com Bruno e Diego antes do início da implementação [09:50 Larissa] |
| **Revisores** | Bruno (Eng. Pleno, Pedidos) · Diego (Eng. Sênior, Plataforma) · Sofia (Segurança) · Marcos (PM) |
| **Documentos** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

## 2. Resumo executivo (TL;DR)

Propomos notificar clientes B2B, via **webhooks de saída assinados**, sempre que o status de um pedido deles mudar, com latência típica **abaixo de 10s**.

O evento é gravado numa **tabela outbox no MySQL, dentro da mesma transação** que muda o status. Um **worker em processo separado** lê essa tabela a cada **2s** e faz o POST HTTPS com **assinatura HMAC-SHA256** (secret por endpoint).

Falhas passam por **5 retentativas com backoff 1m/5m/30m/2h/12h** e depois vão para uma **DLQ**, com replay manual por ADMIN. A garantia é **at-least-once**, e o cliente deduplica pelo **`X-Event-Id`**.

Nenhuma infraestrutura nova: o módulo segue os padrões já existentes do projeto. Estimativa de **três sprints**, com revisão de segurança incluída.

## 3. Contexto e problema

- Atlas Comercial, MaxDistribuição e Nova Cargo fizeram um pedido formal para serem notificados em tempo real sobre mudanças de status dos seus pedidos [09:00 Marcos].
- Hoje eles fazem polling em `GET /orders`, o que é lento e caro para eles.
- A Atlas sinalizou que pode migrar para um concorrente se não houver entrega até o fim do trimestre [09:00 Marcos]. O prazo acertado é o fim de novembro [09:45 Marcos].
- O OMS **não tem nenhum mecanismo de eventos, filas ou notificação externa**.
- A mudança de status (`OrderService.changeStatus` em `src/modules/orders/order.service.ts`) já é uma transação pesada: atualiza `orders`, grava `order_status_history` e mexe em estoque [09:04 Bruno].
- Qualquer solução precisa garantir, ao mesmo tempo:
  - **consistência** (status mudou ⇔ evento existe);
  - **isolamento** (cliente lento ou fora do ar não afeta o fluxo de pedidos).

Escopo: somente **outbound**. Os clientes recebem e não enviam [09:02 Marcos, 09:03 Sofia].

## 4. Proposta técnica

### 4.1 Visão geral

```
 API (src/server.ts)                               Worker (src/worker.ts — novo processo)
 ┌───────────────────────────────┐                ┌─────────────────────────────────────┐
 │ PATCH /orders/:id/status      │                │ loop a cada 2s                      │
 │  └ changeStatus  $transaction │                │  1. lê lote PENDING mais antigo     │
 │     ├ UPDATE orders           │   MySQL        │  2. assina HMAC-SHA256 (secret do   │
 │     ├ INSERT status_history   │ ┌───────────┐  │     endpoint) e faz POST HTTPS 10s  │
 │     ├ stock debit/replenish   │ │ webhook_  │  │  3. 2xx → DELIVERED                 │
 │     └ publishWebhookEvent(tx) ├─► outbox    ◄──┤     falha → retry (backoff) ou DLQ  │
 │ CRUD /webhooks, replay DLQ    │ │ dead_letter│ │  4. registra tentativa em deliveries│
 └───────────────────────────────┘ │ endpoints │  └───────────────┬─────────────────────┘
                                   └───────────┘                  │ HTTPS
                                                                  ▼  Cliente B2B
```

### 4.2 Componentes

1. **Configuração de webhooks (API):** CRUD autenticado para cadastrar URL `https`, escolher os status de interesse e gerenciar a secret, com rotação e grace period de 24h. Inclui consulta ao histórico das últimas entregas [09:31–09:34 Marcos/Bruno].
2. **Publicação transacional:** `publishWebhookEvent(tx, order, fromStatus, toStatus)` é chamada dentro do `changeStatus`. Para cada webhook ativo do cliente que assina o `to_status`, grava um snapshot do evento na outbox. Se nenhum webhook assina, nada é gravado [09:34 Bruno, 09:41 Bruno] → [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md).
3. **Worker de entrega:** processo próprio (`npm run worker`), single-instance, com polling de 2s e timeout HTTP de 10s [09:09 Diego, 09:11 Diego, 09:42 Diego] → [ADR-002](adrs/ADR-002-worker-separado-em-polling.md).
4. **Resiliência:** 5 retentativas com backoff fixo exponencial, DLQ em tabela própria e replay manual restrito a ADMIN, com registro de autoria [09:17 Larissa, 09:18 Diego, 09:36 Sofia] → [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md).
5. **Segurança:** HMAC-SHA256 do corpo em `X-Signature`, secret única por endpoint, TLS obrigatório e limite de 64 KB por payload [09:20–09:24 Sofia/Diego] → [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md).
6. **Semântica de entrega:** at-least-once, com `X-Event-Id` estável entre retentativas e replays [09:24–09:26 Diego] → [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md).
7. **Encaixe no código:** módulo `src/modules/webhooks`, erros `WEBHOOK_*` sobre `AppError`, Pino, error middleware, `requireRole` [09:27–09:30 Bruno/Larissa] → [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md).

Contratos HTTP, modelo de dados, matriz de erros, fluxos passo a passo e a integração arquivo por arquivo estão no [FDD](FDD.md).

## 5. Alternativas consideradas

| # | Alternativa | Trade-off que levou ao descarte |
|---|---|---|
| ALT-01 | **Disparo HTTP síncrono no `changeStatus`** | Um cliente lento prende a transação e trava mudanças de status de outros pedidos. Com o cliente fora do ar, restaria dar rollback no status, o que é inaceitável [09:04 Bruno, 09:06 Diego]. |
| ALT-02 | **Redis Streams / broker dedicado** | Mais infraestrutura para subir e operar (Redis Cluster). É overengineering para um time pequeno, e a outbox no MySQL existente já resolve [09:07 Larissa, 09:07 Diego]. |
| ALT-03 | **Trigger no MySQL para acordar o worker** | O MySQL não tem `LISTEN/NOTIFY`. O trigger só executa SQL, e avisar o processo exigiria improviso. O polling de 2s já atende < 10s [09:09 Bruno, 09:09 Diego]. |
| ALT-04 | **Retry indefinido** ou **só 3 tentativas** | Indefinido deixa eventos pendurados para sempre. Três tentativas cobrem só ~30 min, e já houve manutenção de 2h [09:15 Diego, 09:16 Diego]. |
| ALT-05 | **Exactly-once** | Exige coordenação dos dois lados. At-least-once com `event_id` resolve 99% dos casos e é o padrão de Stripe e GitHub [09:25 Diego]. |
| ALT-06 | **Secret global da plataforma** | Um vazamento compromete todos os clientes [09:21 Sofia]. |
| ALT-07 | **Renderizar o payload no envio** (outbox guarda só `order_id`) | O evento refletiria um estado posterior à mudança [09:52 Larissa]. |

## 6. Questões em aberto

| # | Questão | Origem | Proposta / encaminhamento |
|---|---|---|---|
| Q-01 | **Rate limiting de saída por cliente.** 50 pedidos mudando em um minuto geram 50 chamadas. | [09:38 Diego, 09:39 Larissa] | Fora do escopo. "Observar e decidir depois", com base na métrica de volume por webhook ([FDD §9](FDD.md#9-observabilidade)). |
| Q-02 | **Aviso ao cliente (e-mail) quando o webhook falha repetidamente.** | [09:37 Marcos, 09:37 Larissa] | Adiado para a próxima fase, "depois que a gente medir o impacto". |
| Q-03 | **Escala horizontal do worker** e ordenação. | [09:12 Diego, 09:13 Diego] | Particionar por `order_id` ou usar lock pessimista quando necessário. Hoje é limitação documentada. |
| Q-04 | **Endurecer a autorização do CRUD de webhooks** (hoje qualquer role autenticada). | [09:37 Sofia] | Revisitar em fase futura. |
| Q-05 | **Arquivamento de linhas entregues** (~30 dias). | [09:08 Diego] | Explicitamente fora desta feature. Sem job definido. |
| Q-06 | **"5 tentativas": total ou retentativas?** Os intervalos 1m/5m/30m/2h/12h (≈ 14,6h, "quase 15 horas") implicam 1 envio + 5 retentativas, mas o resumo diz "total 5 tentativas". | [09:17 Diego, 09:48 Larissa] | O FDD adota **1 + 5**, coerente com a janela de ~15h aceita pelo PM [09:17 Marcos]. Confirmar na revisão. |
| Q-07 | **Mecânica do grace period de 24h no envio:** como a secret antiga "fica válida em paralelo" num webhook de saída. | [09:21 Sofia] | Proposta: `X-Signature` com as duas assinaturas durante o grace. Validar na revisão de segurança [09:46 Sofia]. |
| Q-08 | **Ordenação por pedido × backoff:** um evento em retry pode ser ultrapassado por um evento posterior do mesmo pedido, mesmo com single-worker. | Derivada de [09:12 Diego] + [09:17 Larissa] | Hoje: limitação documentada (a garantia é at-least-once, não ordem). Avaliar bloqueio por `(webhook_id, order_id)` na sessão de revisão. |
| Q-09 | **Descoberta de itens da DLQ:** só o replay foi definido, sem endpoint de listagem. | [09:18 Diego] | O ADMIN obtém o `id` por log ou banco. Avaliar `GET /admin/webhooks/dead-letter` com Diego. |

## 7. Impacto e riscos

**Impacto no sistema existente**
- **Caminho crítico alterado:** `changeStatus` passa a ler os webhooks assinantes e gravar na outbox dentro da transação. Se a gravação falhar, o status **não** muda [09:40 Bruno].
- **Operação:** passa a haver um segundo processo (`npm run worker`) com deploy e monitoramento próprios [09:11 Diego].
- **Schema:** três tabelas novas (configuração, outbox, dead letter) mais o histórico de entregas. Nenhuma tabela existente é alterada.
- **Clientes:** precisam validar HMAC e deduplicar por `X-Event-Id`. Marcos documenta no portal [09:26 Marcos, 09:40 Marcos].

**Riscos principais** (detalhe e mitigação completos em [PRD §10](PRD.md#10-riscos-e-mitigação) e [FDD §12](FDD.md#12-riscos-e-mitigação))
- Worker parado → eventos se acumulam (sem perda). Alarme sobre a idade do evento pendente mais antigo.
- Vazamento de secret → rotação self-service e redaction de `*.secret` no Pino [09:22 Diego].
- Prazo apertado (fim de novembro) → escopo mínimo, e revisão de segurança de 2 dias úteis já reservada no plano de 3 sprints [09:46 Larissa, 09:46 Sofia].

## 8. Decisões relacionadas

| ADR | Decisão |
|---|---|
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Padrão Transactional Outbox no MySQL existente |
| [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) | Worker em processo separado, polling de 2s, single-instance |
| [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md) | Retry 1m/5m/30m/2h/12h e DLQ em tabela separada com replay ADMIN |
| [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256, secret por endpoint, rotação com grace de 24h |
| [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) | At-least-once com deduplicação por `X-Event-Id` |
| [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões do projeto (módulo, `AppError`, Pino, middlewares) |
| [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md) | Payload enxuto renderizado como snapshot na inserção |

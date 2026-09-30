# ADR-005 — Garantia de entrega at-least-once com deduplicação via `X-Event-Id`

- **Status:** Aceito
- **Data:** reunião técnica de kickoff (quinta-feira, 09:00)
- **Decisores:** Diego (Plataforma), Larissa (Tech Lead)
- **Consultados:** Sofia (Segurança), Marcos (PM), Bruno (Pedidos)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)

## Contexto

Com outbox + worker + retry ([ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)), algumas situações levam o mesmo evento a ser entregue **mais de uma vez**:

- o cliente processa a requisição, mas a resposta não chega ou demora mais que 10s. O worker registra falha e retenta [09:42 Diego];
- o worker cai depois de enviar e antes de marcar `DELIVERED`, e o evento volta a ser processado;
- um ADMIN faz replay manual de um evento da DLQ [09:18 Diego].

Garantir exactly-once exigiria coordenação entre os dois lados e é bem mais complexo [09:25 Diego].

## Decisão

1. A garantia de entrega é **at-least-once**. O cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado para isso [09:24 Diego].
2. Todo envio leva o header **`X-Event-Id`** com um **UUID gerado quando o evento entra na outbox**, único por evento. Retentativas e replays **reusam o mesmo `event_id`** [09:25 Diego].
3. O cliente **deduplica pelo `event_id`** do lado dele [09:25 Diego, 09:26 Larissa].
4. O `event_id` também vai no corpo (campo `event_id`) [09:43 Diego], então fica coberto pela assinatura HMAC ([ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)).
5. O PM documenta o comportamento com destaque no portal de desenvolvedor [09:26 Marcos].

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Exactly-once** | Exige coordenação bilateral (confirmação em duas fases, estado compartilhado). Muito mais complexo para ganho marginal: "at-least-once com event_id resolve 99% dos casos" [09:25 Diego]. |
| **At-most-once** (envia uma vez, sem retry) | Incompatível com a política de retry do [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md) e com a expectativa dos clientes de não perder atualizações [09:02 Marcos]. |

## Consequências

**Positivas**
- Mesmo modelo de Stripe e GitHub, familiar aos integradores [09:25 Diego].
- O worker pode ser simples: na dúvida, reenvia.
- O `X-Event-Id` também serve de chave de correlação entre os logs do OMS e os do cliente (ver Observabilidade no [FDD](../FDD.md#9-observabilidade)).

**Negativas / trade-offs**
- **Transfere responsabilidade ao cliente**, que precisa guardar os `event_id` já processados [09:25 Sofia].
- Um cliente que ignore o `X-Event-Id` pode processar o mesmo evento em duplicidade. Mitigação: documentação no portal [09:26 Marcos].
- **A ordenação não faz parte da garantia**: at-least-once combinado com retry pode entregar eventos de um mesmo pedido fora de ordem (ver [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)).

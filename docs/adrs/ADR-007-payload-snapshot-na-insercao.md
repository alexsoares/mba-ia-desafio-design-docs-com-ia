# ADR-007 — Payload enxuto, renderizado como snapshot no momento da inserção na outbox

- **Status:** Aceito
- **Data:** reunião técnica de kickoff (pós-reunião, 09:51–09:52)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)

## Contexto

O evento fica na outbox por um tempo que varia de ~2s ([ADR-002](ADR-002-worker-separado-em-polling.md)) até ~15h ([ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)) antes de ser entregue. Nesse intervalo o pedido pode mudar de novo, ou até ser removido: `OrderService.delete` em `src/modules/orders/order.service.ts` apaga pedidos `PENDING` e `CANCELLED`.

Duas perguntas ficaram abertas:

1. O que vai no payload? [09:43 Marcos]
2. O payload é gravado pronto, ou a outbox guarda só o `order_id` e o payload é montado no envio? [09:51 Bruno]

## Decisão

1. **Snapshot na inserção:** o payload JSON é montado e persistido na outbox dentro da transação do `changeStatus`. Ele reflete o estado do pedido **no momento da mudança de status**, mesmo que o pedido mude depois [09:52 Larissa, 09:52 Diego, 09:52 Bruno].
2. **Formato enxuto** [09:43 Diego, 09:44 Bruno]:
   - `event_id` (UUID), `event_type` = `"order.status_changed"`, `timestamp` (ISO 8601);
   - `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`;
   - campos básicos do pedido, como `total_cents`.
   - **Sem `items`.** Quem precisar de detalhes consulta `GET /orders/:id`.
3. **Limite de 64 KB** no payload. Acima disso o evento **não é enviado** e vira erro, **sem truncar** [09:23 Sofia, 09:24 Diego, 09:24 Larissa]. Larissa classificou o limite como requisito não funcional, não como decisão arquitetural separada. O ponto de verificação está detalhado no [FDD](../FDD.md#4-fluxos-detalhados).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Guardar só `order_id` e renderizar no envio** | O evento passaria a refletir um estado posterior ao da mudança ("caso esquisito"), e um pedido removido nem poderia ser renderizado [09:52 Larissa]. |
| **Incluir os itens do pedido no payload** | Infla o payload sem necessidade [09:43 Diego]. |
| **Truncar payloads acima do limite** | Um payload truncado é JSON inválido ou incompleto. "Se chegou nesse tamanho, tem algo errado", então o melhor é errar [09:23 Sofia]. |

## Consequências

**Positivas**
- O evento é imutável e fiel ao momento em que aconteceu. Retentativas e replays enviam exatamente o mesmo corpo, o que mantém a assinatura estável para o mesmo evento e coerente com o `X-Event-Id` ([ADR-005](ADR-005-at-least-once-com-x-event-id.md)).
- O worker não consulta a tabela `orders`: menos carga e nenhum acoplamento com o estado atual do pedido.
- Payload pequeno: com os campos acima, fica muito abaixo de 64 KB [09:24 Diego].

**Negativas / trade-offs**
- Um payload serializado por linha na outbox ocupa mais espaço que só o `order_id`.
- Se o formato mudar no futuro, eventos já enfileirados continuam no formato antigo. Não há versionamento de payload nesta fase.
- O cliente precisa de uma chamada extra (`GET /orders/:id`) se quiser os itens.

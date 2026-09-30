# Architecture Decision Records

Decisões arquiteturais da feature **Sistema de Webhooks de Notificação de Pedidos**, em formato MADR (Status, Contexto, Decisão, Alternativas Consideradas, Consequências). Arquivos nomeados `ADR-NNN-titulo-em-kebab-case.md`.

| ADR | Decisão | Status |
|---|---|---|
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Transactional Outbox no MySQL existente | Aceito |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Worker em processo separado, polling de 2s | Aceito |
| [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md) | Retry com backoff 1m/5m/30m/2h/12h e DLQ em tabela separada | Aceito |
| [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256, secret por endpoint, rotação com grace de 24h | Aceito |
| [ADR-005](ADR-005-at-least-once-com-x-event-id.md) | Entrega at-least-once com deduplicação via `X-Event-Id` | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões existentes do projeto | Aceito |
| [ADR-007](ADR-007-payload-snapshot-na-insercao.md) | Payload enxuto renderizado como snapshot na inserção | Aceito |

Proposta consolidada: [RFC](../RFC.md) · Implementação: [FDD](../FDD.md) · Origem de cada item: [TRACKER](../TRACKER.md)

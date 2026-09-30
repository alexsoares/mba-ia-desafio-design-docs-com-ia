# ADR-003 — Retry com backoff exponencial (1m/5m/30m/2h/12h) e DLQ em tabela separada

- **Status:** Aceito
- **Data:** reunião técnica de kickoff (quinta-feira, 09:00)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

## Contexto

O endpoint do cliente pode estar lento, fora do ar ou em manutenção planejada. Já houve cliente com duas horas de indisponibilidade em manutenção [09:16 Diego]. O sistema precisa:

- insistir o bastante para cobrir indisponibilidades reais;
- não deixar eventos "pendurados para sempre" quando o cliente some de vez [09:15 Diego];
- guardar o que falhou para depuração e reprocessamento [09:18 Diego].

Uma chamada que não responde em **10 segundos** conta como falha e vai para retry [09:42 Diego].

## Decisão

1. **Backoff exponencial com intervalos fixos:** 1 min → 5 min → 30 min → 2 h → 12 h [09:17 Diego, 09:17 Larissa].
2. **Teto de 5 retentativas.** Interpretação adotada: **1 envio inicial + 5 retentativas**, ou seja, até 6 chamadas HTTP por evento. É a única leitura coerente com os 5 intervalos e com a janela de "quase 15 horas entre primeira falha e última tentativa" (1+5+30+120+720 min = 876 min ≈ 14,6 h) [09:17 Diego]. O resumo final fala em "total 5 tentativas" [09:48 Larissa], o que tornaria a janela ≈ 2,6 h. A divergência está registrada como questão em aberto no [RFC](../RFC.md#6-questões-em-aberto).
3. Esgotadas as retentativas, o evento é marcado `FAILED` na outbox e copiado para uma **DLQ em tabela separada, `webhook_dead_letter`**, com payload, motivo da falha e timestamp [09:18 Diego, 09:48 Larissa].
4. **Reprocessamento manual** via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente [09:18 Diego]. O endpoint exige **role `ADMIN`** e registra quem fez o replay [09:36 Sofia, 09:36 Larissa].

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Retry indefinido com backoff** | O evento fica pendurado para sempre se o cliente sumiu [09:15 Diego]. |
| **3 tentativas (mais agressivo)** | Cobre só ~30 min. Uma indisponibilidade de manhã mataria o evento, e já houve cliente fora por 2 h em manutenção planejada [09:16 Bruno, 09:16 Diego]. |
| **Marcar como `failed` na própria outbox, sem tabela de DLQ** | Suja a leitura da outbox principal. A tabela separada fica "mais limpa" e serve de evidência para debug e reprocessamento [09:17 Larissa, 09:18 Diego]. |

## Consequências

**Positivas**
- Cobre indisponibilidades de até ~15 h, e o PM considera aceitável perder a entrega automática além disso [09:17 Marcos].
- O teto evita acúmulo infinito de eventos inúteis.
- A DLQ separada mantém a outbox enxuta e cria um ponto único de investigação e replay.

**Negativas / trade-offs**
- **Latência máxima de entrega de ~14,6 h** para um evento que só teve sucesso na última retentativa. O cliente precisa lidar com eventos atrasados.
- **O backoff pode inverter a ordem por pedido:** se o evento `PAID` de um pedido estiver aguardando retry e o `PROCESSING` do mesmo pedido chegar depois, o segundo pode ser entregue antes. Isso reforça que a ordenação não é garantida (ver [ADR-002](ADR-002-worker-separado-em-polling.md)). Registrado como questão em aberto no RFC.
- Replay é manual: não há reprocessamento automático da DLQ nem aviso proativo ao cliente. O aviso por e-mail foi adiado para a próxima fase [09:37 Larissa].
- Não foi definido endpoint para **listar** a DLQ. O ADMIN precisa obter o `id` por outro meio (logs/banco). Questão em aberto.

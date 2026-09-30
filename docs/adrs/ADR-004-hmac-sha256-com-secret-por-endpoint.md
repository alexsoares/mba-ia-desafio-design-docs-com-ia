# ADR-004 — Autenticação por HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h

- **Status:** Aceito (implementação sujeita à revisão de segurança da Sofia antes do deploy)
- **Data:** reunião técnica de kickoff (quinta-feira, 09:00)
- **Decisores:** Sofia (Segurança), Larissa (Tech Lead)
- **Consultados:** Bruno, Diego
- **Relacionados:** [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)

## Contexto

Os webhooks são **somente de saída** (outbound): o OMS envia e o cliente recebe [09:02 Marcos, 09:03 Sofia]. Eles levam dados de pedidos para endpoints fora da nossa infraestrutura. O cliente precisa conseguir verificar que:

1. a requisição veio de fato do OMS (autenticidade);
2. o payload não foi adulterado no caminho (integridade) [09:19 Sofia].

Já houve cliente que vazou secret em log da própria aplicação [09:22 Diego], então o raio de impacto de um vazamento precisa ser pequeno e a troca de secret precisa ser possível sem downtime.

## Decisão

1. **Assinatura HMAC-SHA256 sobre o corpo do request**, enviada no header `X-Signature` [09:20 Sofia, 09:22 Sofia]. O cálculo usa o módulo nativo `node:crypto`, sem dependência nova.
2. **Uma secret por endpoint de webhook**, nunca uma secret global da plataforma: "se vaza uma, vaza tudo" [09:21 Sofia].
3. **A secret é gerada pelo OMS** e devolvida ao cliente na criação do webhook [09:31 Marcos]. A configuração do webhook guarda `url`, `secret`, `customer_id` e o estado ativo [09:21 Bruno, 09:21 Sofia].
4. **Rotação via API:** o cliente pede uma nova secret por um endpoint. A antiga continua **válida em paralelo por 24h** e depois é descartada [09:21 Sofia, 09:22 Sofia].
5. **TLS obrigatório:** a URL precisa ser `https`. Uma URL `http` é recusada com erro de validação no schema Zod. Isso não é tratado como decisão arquitetural, e sim como validação [09:23 Sofia].
6. O código de HMAC e de geração de secret passa por **revisão de segurança dedicada (≥ 2 dias úteis)** antes do deploy [09:46 Sofia].

### Como a "validade em paralelo" funciona em webhook de saída (proposta, a validar com Sofia)

Como é o OMS quem assina, "a antiga fica válida" só faz sentido se o cliente conseguir validar mensagens assinadas com qualquer uma das duas secrets. **Proposta** para a revisão de segurança: durante o grace period, o `X-Signature` carrega as duas assinaturas, separadas por vírgula (`sha256=<hmac_nova>,sha256=<hmac_antiga>`). O cliente aceita a mensagem se qualquer uma bater. Registrado como questão em aberto no [RFC](../RFC.md#6-questões-em-aberto).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Secret global da plataforma** | Um vazamento compromete todos os clientes [09:21 Sofia]. |
| **Rotação sem grace period** (a antiga morre na hora) | O cliente não tem tempo de migrar os sistemas e perde eventos entre a rotação e o deploy dele [09:21 Sofia]. |
| **Aceitar URL `http`** | O payload trafegaria em texto claro. Descartado por validação [09:23 Sofia]. |

## Consequências

**Positivas**
- Padrão de mercado: "todo cliente sério tem biblioteca pra isso" [09:20 Sofia].
- Vazamento isolado por endpoint, com remediação self-service (rotação) sem downtime.
- Sem dependência externa: HMAC e geração de secret usam `node:crypto`.

**Negativas / trade-offs**
- **A secret precisa ser armazenada recuperável** (não dá para guardar só o hash), porque o worker precisa dela para assinar. Criptografia em repouso **não foi discutida** e fica como ponto da revisão de segurança.
- A secret **não pode aparecer em logs**. `src/shared/logger/index.ts` precisa ganhar o caminho `*.secret` no `redact` (hoje redige `*.password`, `*.token`, etc.).
- Durante as 24h de grace, o worker assina duas vezes por envio (custo desprezível).
- A assinatura cobre só o corpo, e o `X-Timestamp` fica fora dela. O `timestamp` do evento dentro do corpo é assinado, mas o do header (instante do envio) não é. A proteção contra replay fica a critério do cliente [09:44 Diego].

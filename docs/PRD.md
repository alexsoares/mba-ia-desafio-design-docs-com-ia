# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Envolvidos** | Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |
| **Status** | Aprovado na reunião de kickoff. Prazo alvo: fim de novembro [09:45 Marcos] |
| **Documentos** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

> Os itens têm IDs (`PRD-…`), e a origem de cada um está em [TRACKER.md](TRACKER.md). Os timestamps `[hh:mm Nome]` apontam para a `TRANSCRICAO.md`.

## 1. Resumo e contexto da feature

O OMS vai **notificar automaticamente os sistemas dos clientes B2B sempre que o status de um pedido deles mudar**, com um POST HTTPS (webhook) assinado, enviado em poucos segundos. O cliente cadastra a URL que quer receber e escolhe quais status o interessam. O OMS cuida da entrega, das retentativas e da segurança.

Hoje não existe mecanismo de notificação externa no OMS. O ciclo de vida do pedido (`PENDING → PAID → PROCESSING → SHIPPED → DELIVERED`, com `CANCELLED` possível até `PROCESSING`) já está implementado e auditado (`src/modules/orders/order.status.ts`). Falta só avisar o cliente.

## 2. Problema e motivação

- **Problema:** para saber se um pedido mudou, os clientes consultam `GET /orders` periodicamente. Isso torna a integração **lenta e cara** para eles [09:00 Marcos].
- **Demanda formal:** três clientes B2B, Atlas Comercial, MaxDistribuição e Nova Cargo, pediram notificação em tempo real [09:00 Marcos].
- **Risco comercial:** a Atlas indicou que pode **migrar para um concorrente** se a funcionalidade não for entregue até o fim do trimestre [09:00 Marcos].
- **Expectativa de "tempo real":** para os clientes, qualquer coisa **abaixo de 10 segundos** já conta. O importante é não precisar atualizar manualmente [09:02 Marcos].

## 3. Público-alvo e cenários de uso

**Personas**

| Persona | Descrição |
|---|---|
| **Integrador do cliente B2B** | Time técnico de Atlas, MaxDistribuição, Nova Cargo e futuros clientes. Recebe os webhooks no sistema próprio. |
| **Usuário operador que representa o cliente** | Usuário do OMS, autenticado por JWT, que cadastra e mantém os webhooks do cliente pela API [09:32 Marcos]. |
| **Administrador do OMS (role `ADMIN`)** | Investiga falhas permanentes e reprocessa eventos da DLQ [09:36 Sofia]. |

**Cenários**

| ID | Cenário |
|---|---|
| PRD-CEN-01 | A Nova Cargo só quer saber quando o pedido é **enviado** ou **entregue**. Cadastra um webhook com `events = [SHIPPED, DELIVERED]` e não recebe as demais mudanças [09:33 Marcos]. |
| PRD-CEN-02 | A Atlas está em **manutenção planejada por 2 horas**. Os eventos do período são reenviados automaticamente até o sistema voltar, sem perda [09:16 Diego]. |
| PRD-CEN-03 | O integrador **vazou a secret** num log da aplicação dele. Ele pede uma nova secret pela API, e a antiga continua válida por 24h enquanto ele atualiza os sistemas [09:21 Sofia, 09:22 Diego]. |
| PRD-CEN-04 | O cliente diz que "não recebeu" um evento. O operador consulta o **histórico das últimas 100 entregas** (status, resposta, tempo) para diagnosticar [09:34 Marcos]. |
| PRD-CEN-05 | Um endpoint ficou fora do ar mais de ~15 horas e o evento foi para a DLQ. Após a correção, um **ADMIN reprocessa** o evento manualmente [09:18 Diego]. |
| PRD-CEN-06 | O cliente tem **vários endpoints** cadastrados e identifica pelo header `X-Webhook-Id` qual cadastro originou o envio [09:44 Sofia]. |

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta |
|---|---|---|---|
| PRD-OBJ-01 | Notificar em "tempo real" | Latência entre a mudança de status e a 1ª tentativa de entrega | **p95 < 10 s** (projeto: ≤ 2 s de polling + ≤ 10 s de timeout) [09:02 Marcos, 09:10 Larissa] |
| PRD-OBJ-02 | Não perder eventos | % de mudanças de status (com webhook assinante) que geram evento na outbox | **100%**, garantido pela transação [09:40 Bruno] |
| PRD-OBJ-03 | Tolerar indisponibilidades do cliente | Janela coberta por retentativas automáticas antes da DLQ | **≈ 15 h** (1m/5m/30m/2h/12h) [09:17 Diego, 09:17 Marcos] |
| PRD-OBJ-04 | Reter os clientes que pediram a feature | Clientes solicitantes integrados em produção | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) até **fim de novembro** [09:45 Marcos, 09:47 Marcos] |
| PRD-OBJ-05 | Não degradar o fluxo de pedidos | Mudanças de status afetadas por lentidão ou queda de clientes | **0**. Nenhuma chamada externa na transação [09:04 Bruno] |

## 5. Escopo

### 5.1 Incluso

- Webhooks **de saída** (OMS → cliente) para o evento `order.status_changed` [09:02 Marcos, 09:03 Sofia].
- API para cadastrar, listar, editar e remover webhooks, com filtro por status [09:31–09:33].
- Rotação de secret com grace period de 24h [09:21 Sofia].
- Histórico de entregas por webhook [09:34 Marcos].
- Retentativas automáticas, DLQ e reprocessamento manual por ADMIN [09:17–09:18, 09:36].
- Assinatura HMAC-SHA256, HTTPS obrigatório e limite de 64 KB [09:20–09:24].

### 5.2 Fora de escopo

| ID | Item | Situação na reunião |
|---|---|---|
| PRD-FE-01 | **Aviso por e-mail** ao cliente quando o webhook falha repetidamente | **Adiado** para a próxima fase, "depois que a gente medir o impacto" [09:37 Larissa, 09:38 Marcos] |
| PRD-FE-02 | **Dashboard/painel visual** para o cliente ver seus webhooks | **Descartado** nesta feature. É projeto separado do time de frontend, e aqui "só endpoints" [09:40 Larissa] |
| PRD-FE-03 | **Rate limiting** de envio por cliente | **Não decidido**: "observar e decidir depois" [09:39 Diego, 09:39 Larissa] |
| PRD-FE-04 | **Webhooks de entrada** (cliente → OMS) | **Descartado**: "Eles querem receber, não mandar" [09:02 Marcos] |
| PRD-FE-05 | **Garantia de ordenação global** e múltiplos workers | **Adiado**: limitação conhecida, ordem só por pedido e com um único worker [09:13 Larissa, 09:14 Marcos] |
| PRD-FE-06 | **Arquivamento** de eventos entregues (~30 dias) | **Fora do escopo** desta feature [09:08 Diego] |
| PRD-FE-07 | **Exactly-once** (entrega exatamente uma vez) | **Descartado** em favor de at-least-once [09:25 Diego] |
| PRD-FE-08 | **Itens do pedido** no payload | **Descartado**: payload enxuto, detalhes via `GET /orders/:id` [09:43 Diego] |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-FR-01 | Enviar notificação ao cliente sempre que o status de um pedido dele mudar | [09:00 Marcos] |
| PRD-FR-02 | Cadastrar webhook (`POST`) informando `url`, lista de status desejados e `customerId`. O `customerId` vai no body ou no path, **não** vem do JWT | [09:31 Marcos, 09:32 Larissa] |
| PRD-FR-03 | A secret é **gerada pelo OMS** e devolvida na criação | [09:31 Marcos] |
| PRD-FR-04 | Editar (`PATCH`), remover (`DELETE`) e listar (`GET`) os webhooks de um cliente | [09:33 Bruno] |
| PRD-FR-05 | Cada webhook escolhe quais status quer receber. Sem assinante do status, nenhum evento é gerado | [09:33 Marcos, 09:34 Bruno] |
| PRD-FR-06 | Rotacionar a secret pela API. A antiga segue válida por 24h e depois é descartada | [09:21 Sofia] |
| PRD-FR-07 | Consultar o histórico das **últimas 100 entregas** de um webhook: sucesso/falha, payload, resposta e tempo de resposta (`GET /webhooks/:id/deliveries`) | [09:34 Marcos] |
| PRD-FR-08 | Retentar automaticamente entregas com falha, com intervalos crescentes: 1 min, 5 min, 30 min, 2 h, 12 h | [09:17 Larissa] |
| PRD-FR-09 | Mover para a DLQ os eventos que esgotarem as retentativas, guardando payload, motivo e data | [09:18 Diego] |
| PRD-FR-10 | Reprocessar manualmente um evento da DLQ (`POST /admin/webhooks/dead-letter/:id/replay`), **somente ADMIN**, registrando quem executou | [09:18 Diego, 09:36 Sofia] |
| PRD-FR-11 | Cada envio leva um identificador único do evento (`X-Event-Id`) para o cliente deduplicar | [09:25 Diego] |
| PRD-FR-12 | Cada envio é assinado (HMAC-SHA256, `X-Signature`) com a secret daquele endpoint | [09:20 Sofia, 09:21 Sofia] |
| PRD-FR-13 | Cada envio informa o cadastro de origem (`X-Webhook-Id`) e o instante do envio (`X-Timestamp`) | [09:44 Diego, 09:44 Sofia] |
| PRD-FR-14 | O conteúdo da notificação traz: `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`. **Sem itens**. O conteúdo reflete o pedido no momento da mudança | [09:43 Diego, 09:52 Larissa] |
| PRD-FR-15 | O CRUD de webhooks fica disponível a qualquer usuário autenticado | [09:36 Marcos, 09:37 Sofia] |

## 7. Requisitos não funcionais

| ID | Categoria | Requisito | Origem |
|---|---|---|---|
| PRD-NFR-01 | Latência | Notificação em < 10 s. O worker verifica eventos a cada 2 s, e o pior caso mínimo de 2 s é aceito | [09:02 Marcos, 09:10 Larissa] |
| PRD-NFR-02 | Consistência | Se o status mudou, o evento existe. Se a mudança falhou, o evento não existe (atomicidade) | [09:06 Diego, 09:40 Bruno] |
| PRD-NFR-03 | Isolamento | Cliente lento ou fora do ar não afeta mudanças de status | [09:04 Bruno] |
| PRD-NFR-04 | Entrega | At-least-once. O cliente pode receber duplicatas e deve deduplicar por `X-Event-Id` | [09:24 Diego, 09:26 Larissa] |
| PRD-NFR-05 | Timeout | Uma tentativa sem resposta em 10 s conta como falha e vai para retry | [09:42 Diego] |
| PRD-NFR-06 | Segurança | URL precisa ser `https`. `http` é recusado com erro de validação | [09:23 Sofia] |
| PRD-NFR-07 | Segurança | Secret única por endpoint, nunca global | [09:21 Sofia] |
| PRD-NFR-08 | Tamanho | Payload limitado a 64 KB. Acima disso o evento não é enviado e é tratado como erro (não trunca) | [09:23 Sofia, 09:24 Larissa] |
| PRD-NFR-09 | Ordenação | Ordem garantida apenas por pedido e só com um único worker. Não há ordenação global | [09:13 Larissa] |
| PRD-NFR-10 | Disponibilidade | O processo de entrega é independente da API: restart da API não para as entregas | [09:11 Diego] |
| PRD-NFR-11 | Manutenibilidade | Seguir os padrões do projeto (módulos, erros `WEBHOOK_*`, logger Pino), sem infraestrutura nova | [09:30 Larissa, 09:07 Diego] |
| PRD-NFR-12 | Auditoria | Replay de DLQ registra o autor | [09:36 Sofia] |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off aceito | ADR |
|---|---|---|
| Outbox no MySQL existente, na mesma transação da mudança de status | Latência mínima de até 2 s, em troca de consistência e nenhuma infra nova | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Worker separado em polling de 2 s | Dois processos para operar. Ordem garantida só com um worker | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 retentativas (1m→12h) + DLQ + replay manual | Eventos podem chegar até ~15 h depois. Sem reprocessamento automático | [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md) |
| HMAC-SHA256, secret por endpoint, rotação com 24 h | A secret precisa ser guardada de forma recuperável para assinar | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| At-least-once com `X-Event-Id` | O cliente precisa deduplicar | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Reuso dos padrões do projeto | Ganha-se velocidade, mas as regras de autorização herdam a permissividade atual | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| Payload enxuto, fotografado no momento da mudança | O cliente faz uma chamada extra se quiser os itens | [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md) |

## 9. Dependências

| ID | Dependência | Origem |
|---|---|---|
| PRD-DEP-01 | **Revisão de segurança** da Sofia, com pelo menos 2 dias úteis antes do deploy, focada em HMAC e geração de secret | [09:46 Sofia, 09:49 Sofia] |
| PRD-DEP-02 | **Portal de desenvolvedor** atualizado por Marcos: integração via API, validação de assinatura e deduplicação por `X-Event-Id` | [09:26 Marcos, 09:40 Marcos] |
| PRD-DEP-03 | **Confirmação de prazo** com a Atlas e atualização dos três clientes | [09:47 Marcos, 09:49 Marcos] |
| PRD-DEP-04 | Endpoints HTTPS disponíveis do lado dos clientes | [09:23 Sofia] |
| PRD-DEP-05 | Fluxo de mudança de status existente (`OrderService.changeStatus`) como ponto de origem dos eventos | [09:40 Bruno] · `src/modules/orders/order.service.ts` |
| PRD-DEP-06 | **Sessão de revisão do design** com Bruno e Diego antes de iniciar o código | [09:50 Larissa] |

**Cronograma (estimativa da Tech Lead [09:46 Larissa]) — três sprints, com revisão de segurança incluída no fim [09:47 Larissa]:**

| Etapa | Estimativa |
|---|---|
| Modelagem de outbox e DLQ | 1 sprint |
| Worker e retry | 1 sprint |
| CRUD de configuração e histórico de entregas | ½ sprint |
| Integração no `order.service` e testes ponta a ponta | ½ sprint |
| HMAC, schemas e validações | "mais um pouco" |

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| PRD-RISK-01 | **Perder a Atlas** por atraso na entrega (prazo: fim de novembro) | Média | Alto | Escopo mínimo: dashboard, e-mail e rate limit fora. Estimativa de 3 sprints, e Marcos confirma prazo com o cliente [09:00 Marcos, 09:45 Marcos, 09:47 Marcos] |
| PRD-RISK-02 | **Cliente processa o mesmo evento duas vezes** (at-least-once) | Média | Médio | `X-Event-Id` em todo envio e documentação destacada no portal [09:25 Diego, 09:26 Marcos] |
| PRD-RISK-03 | **Vazamento de secret** do cliente | Baixa | Alto | Secret por endpoint (raio de impacto limitado) e rotação self-service com grace de 24h [09:21 Sofia, 09:22 Diego] |
| PRD-RISK-04 | **Cliente fora do ar por mais de ~15 h** perde entregas automáticas | Baixa | Médio | DLQ e replay manual por ADMIN. O PM considera aceitável [09:17 Marcos, 09:18 Diego] |
| PRD-RISK-05 | **Cliente sobrecarregado** por rajadas de eventos (sem rate limit) | Média | Médio | Monitorar volume por webhook e decidir depois [09:38 Diego, 09:39 Larissa] |
| PRD-RISK-06 | **Eventos fora de ordem** para o mesmo pedido | Baixa | Baixo | Limitação documentada. Cada evento traz `from_status`/`to_status`, e os clientes não pedem ordenação global [09:13 Larissa, 09:14 Marcos] |

## 11. Critérios de aceitação

| ID | Critério |
|---|---|
| PRD-CA-01 | Dado um webhook ativo que assina `SHIPPED`, quando o pedido do cliente passa para `SHIPPED`, o endpoint recebe um POST HTTPS em menos de 10 s com o conteúdo do PRD-FR-14 |
| PRD-CA-02 | Dado um webhook que assina só `DELIVERED`, quando o pedido vai para `PAID`, nenhuma notificação é enviada |
| PRD-CA-03 | Cadastrar webhook com URL `http://` retorna erro de validação |
| PRD-CA-04 | A secret aparece apenas na resposta de criação e de rotação |
| PRD-CA-05 | O cliente consegue validar a assinatura `X-Signature` com a secret recebida |
| PRD-CA-06 | Após rotacionar a secret, envios nas 24 h seguintes continuam verificáveis com a secret antiga (mecânica exata em validação — [RFC Q-07](RFC.md#6-questões-em-aberto)) |
| PRD-CA-07 | Com o endpoint do cliente fora do ar, o evento é retentado em 1m, 5m, 30m, 2h e 12h e depois vai para a DLQ |
| PRD-CA-08 | Um ADMIN reprocessa um evento da DLQ, e um usuário OPERATOR recebe 403 ao tentar |
| PRD-CA-09 | Reenvios do mesmo evento chegam com o mesmo `X-Event-Id` |
| PRD-CA-10 | O histórico de entregas mostra as últimas 100 tentativas com sucesso/falha, payload, resposta e tempo |
| PRD-CA-11 | Parar a API não interrompe entregas pendentes, e parar o worker não impede mudanças de status |
| PRD-CA-12 | Falha ao registrar o evento impede a mudança de status: nenhum status muda sem evento |

## 12. Estratégia de testes e validação

- **Integração (Vitest + Supertest, MySQL real):** mesmo padrão de `tests/orders.test.ts` e `tests/helpers/factories.ts`. Cobre CRUD, filtro por status, atomicidade com `changeStatus`, autorização ADMIN no replay e validação `https`.
- **Worker:** um servidor HTTP local faz o papel do cliente e simula 2xx, 5xx, lentidão acima de 10 s e queda. Verifica headers, assinatura, agenda de retry e ida para a DLQ (critérios técnicos no [FDD §13](FDD.md#13-critérios-de-aceite-técnicos)).
- **Segurança:** revisão dedicada da Sofia, com ≥ 2 dias úteis, sobre HMAC, geração e rotação de secret [09:46 Sofia].
- **Revisão de design:** sessão com Bruno e Diego antes do código [09:50 Larissa].
- **Validação com clientes:** Marcos confirma prazo e comunica os três clientes, e a documentação do portal orienta a integração [09:47 Marcos, 09:40 Marcos].
- **Pós-lançamento:** acompanhar a latência p95 (PRD-OBJ-01), o volume da DLQ e o volume por cliente. Os dados alimentam as decisões adiadas sobre rate limit e e-mail [09:37 Larissa, 09:39 Larissa].

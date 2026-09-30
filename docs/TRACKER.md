# Tracker de Rastreabilidade — Sistema de Webhooks de Notificação de Pedidos

Este tracker liga cada item registrado em [PRD](PRD.md), [RFC](RFC.md), [FDD](FDD.md) e [ADRs](adrs/) à sua origem:

- **TRANSCRICAO**: `TRANSCRICAO.md`, localizada como `[hh:mm] Falante`;
- **CODIGO**: caminho real do arquivo no repositório.

Quando um item tem mais de uma origem, a linha aponta a **fala decisiva** (a que fechou o ponto). Itens marcados como *escolha de implementação* não foram fixados na reunião e foram derivados pelo autor do FDD. Nesses casos, a linha aponta a fala ou o arquivo que **motivou** a escolha, para que o revisor possa contestá-la.

**Legenda de tipos:** Contexto · Requisito Funcional · Requisito Não Funcional · Decisão · Restrição · Trade-off · Alternativa Descartada · Fora de Escopo · Questão em Aberto · Risco · Dependência · Métrica · Contrato · Erro · Integração · Critério de Aceite · Cenário · Escolha de Implementação

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Atlas, MaxDistribuição e Nova Cargo pediram notificação em tempo real de mudança de status | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Hoje os clientes fazem polling em GET /orders, integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Risco | Atlas pode migrar para concorrente se não houver entrega até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | docs/PRD.md | Contexto | "Tempo real" para os clientes = abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-05 | docs/PRD.md | Contexto | Ciclo de vida PENDING→PAID→PROCESSING→SHIPPED→DELIVERED, CANCELLED até PROCESSING | CODIGO | src/modules/orders/order.status.ts |
| PRD-CEN-01 | docs/PRD.md | Cenário | Cliente assina só SHIPPED e DELIVERED | TRANSCRICAO | [09:33] Marcos |
| PRD-CEN-02 | docs/PRD.md | Cenário | Cliente em manutenção planejada de 2h recebe via retry | TRANSCRICAO | [09:16] Diego |
| PRD-CEN-03 | docs/PRD.md | Cenário | Cliente vazou secret e rotaciona com 24h de grace | TRANSCRICAO | [09:22] Diego |
| PRD-CEN-04 | docs/PRD.md | Cenário | Operador consulta últimas 100 entregas para diagnóstico | TRANSCRICAO | [09:34] Marcos |
| PRD-CEN-05 | docs/PRD.md | Cenário | ADMIN reprocessa evento da DLQ após correção do cliente | TRANSCRICAO | [09:18] Diego |
| PRD-CEN-06 | docs/PRD.md | Cenário | Cliente com vários endpoints identifica o cadastro por X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| PRD-PER-01 | docs/PRD.md | Contexto | Persona: usuário do OMS que representa o cliente e cadastra via API com JWT | TRANSCRICAO | [09:32] Marcos |
| PRD-PER-02 | docs/PRD.md | Contexto | Persona: ADMIN responsável por replay de DLQ | TRANSCRICAO | [09:36] Sofia |
| PRD-OBJ-01 | docs/PRD.md | Métrica | p95 < 10s entre mudança de status e 1ª tentativa de entrega | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Métrica | 100% das mudanças com assinante geram evento (transação) | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-03 | docs/PRD.md | Métrica | Janela de ~15h de retentativas antes da DLQ | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-04 | docs/PRD.md | Métrica | 3 de 3 clientes solicitantes integrados até fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-05 | docs/PRD.md | Métrica | 0 mudanças de status afetadas por clientes lentos/offline | TRANSCRICAO | [09:04] Bruno |
| PRD-INC-01 | docs/PRD.md | Restrição | Escopo apenas outbound (OMS → cliente) | TRANSCRICAO | [09:03] Sofia |
| PRD-FE-01 | docs/PRD.md | Fora de Escopo | E-mail de aviso em falhas repetidas adiado para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-FE-02 | docs/PRD.md | Fora de Escopo | Dashboard visual é projeto separado do frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-FE-03 | docs/PRD.md | Fora de Escopo | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-FE-04 | docs/PRD.md | Fora de Escopo | Webhooks de entrada descartados | TRANSCRICAO | [09:02] Marcos |
| PRD-FE-05 | docs/PRD.md | Fora de Escopo | Ordenação global e múltiplos workers adiados (limitação conhecida) | TRANSCRICAO | [09:13] Larissa |
| PRD-FE-06 | docs/PRD.md | Fora de Escopo | Arquivamento de entregues (~30 dias) fora desta feature | TRANSCRICAO | [09:08] Diego |
| PRD-FE-07 | docs/PRD.md | Fora de Escopo | Exactly-once descartado | TRANSCRICAO | [09:25] Diego |
| PRD-FE-08 | docs/PRD.md | Fora de Escopo | Items do pedido não entram no payload | TRANSCRICAO | [09:43] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Notificar cliente a cada mudança de status de pedido | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | POST para cadastrar webhook com url, status desejados e customerId | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02b | docs/PRD.md | Restrição | customerId vem no body/path, não do JWT (JWT é do operador) | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Secret gerada pelo OMS e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | PATCH, DELETE e GET de webhooks do customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por lista de status | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Rotação de secret via API com grace de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | GET /webhooks/:id/deliveries com últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Retry automático 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | DLQ com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Replay manual da DLQ via endpoint admin | TRANSCRICAO | [09:18] Diego |
| PRD-FR-10b | docs/PRD.md | Restrição | Replay exige ADMIN e registra autor | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | X-Event-Id em todo envio para dedup | TRANSCRICAO | [09:25] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 em X-Signature | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Headers X-Webhook-Id e X-Timestamp | TRANSCRICAO | [09:44] Diego |
| PRD-FR-13b | docs/PRD.md | Requisito Funcional | Header X-Webhook-Id sugerido para clientes com vários endpoints | TRANSCRICAO | [09:44] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Campos do payload (event_id, event_type, timestamp, order_id, order_number, from/to_status, customer_id, total_cents) | TRANSCRICAO | [09:43] Diego |
| PRD-FR-14b | docs/PRD.md | Decisão | Payload reflete o pedido no momento da mudança (snapshot) | TRANSCRICAO | [09:52] Larissa |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | CRUD de webhooks liberado para qualquer usuário autenticado | TRANSCRICAO | [09:37] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência < 10s, polling 2s, pior caso mínimo de 2s aceito | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Atomicidade status/evento | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Cliente lento não afeta mudanças de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Entrega at-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por tentativa | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | URL obrigatoriamente https, http recusado | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Limite de 64 KB, erro se ultrapassar, sem truncar | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Ordenação só por pedido e com single-worker | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Entrega independente do processo da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Seguir padrões do projeto, sem infra nova | TRANSCRICAO | [09:30] Larissa |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Auditoria do autor do replay | TRANSCRICAO | [09:36] Sofia |
| PRD-DEC-01 | docs/PRD.md | Trade-off | Tabela-resumo de decisões e trade-offs (espelha ADR-001..007) | TRANSCRICAO | [09:48] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança de ≥ 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-01b | docs/PRD.md | Dependência | Lembrete de agendar a revisão de segurança antes de subir | TRANSCRICAO | [09:49] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Portal de desenvolvedor documenta dedup por event_id | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-02b | docs/PRD.md | Dependência | Portal de desenvolvedor documenta integração via API | TRANSCRICAO | [09:40] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Marcos confirma prazo com a Atlas | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-03b | docs/PRD.md | Dependência | Marcos atualiza os clientes na mesma tarde | TRANSCRICAO | [09:49] Marcos |
| PRD-DEP-04 | docs/PRD.md | Dependência | Clientes precisam de endpoint HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-DEP-05 | docs/PRD.md | Dependência | Origem dos eventos é OrderService.changeStatus | CODIGO | src/modules/orders/order.service.ts |
| PRD-DEP-06 | docs/PRD.md | Dependência | Sessão de revisão do design com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |
| PRD-CRON-01 | docs/PRD.md | Restrição | Estimativa de 3 sprints (outbox/DLQ 1, worker/retry 1, CRUD ½, integração ½, HMAC) | TRANSCRICAO | [09:46] Larissa |
| PRD-CRON-02 | docs/PRD.md | Restrição | 3 sprints com revisão de segurança incluída no fim | TRANSCRICAO | [09:47] Larissa |
| PRD-RISK-01 | docs/PRD.md | Risco | Perder a Atlas por atraso (prob. média, impacto alto) | TRANSCRICAO | [09:45] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Cliente processa duplicata (at-least-once) | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-03 | docs/PRD.md | Risco | Vazamento de secret do cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Cliente fora > 15h perde entrega automática (aceito pelo PM) | TRANSCRICAO | [09:17] Marcos |
| PRD-RISK-05 | docs/PRD.md | Risco | Rajada de eventos sobrecarrega cliente (sem rate limit) | TRANSCRICAO | [09:38] Diego |
| PRD-RISK-06 | docs/PRD.md | Risco | Eventos fora de ordem por pedido | TRANSCRICAO | [09:14] Marcos |
| PRD-CA-01 | docs/PRD.md | Critério de Aceite | Webhook assinante recebe POST HTTPS em < 10s | TRANSCRICAO | [09:02] Marcos |
| PRD-CA-02 | docs/PRD.md | Critério de Aceite | Status não assinado não gera notificação | TRANSCRICAO | [09:34] Bruno |
| PRD-CA-03 | docs/PRD.md | Critério de Aceite | URL http retorna erro de validação | TRANSCRICAO | [09:23] Sofia |
| PRD-CA-04 | docs/PRD.md | Critério de Aceite | Secret só na criação e rotação | TRANSCRICAO | [09:31] Marcos |
| PRD-CA-05 | docs/PRD.md | Critério de Aceite | Cliente valida X-Signature com a secret | TRANSCRICAO | [09:20] Sofia |
| PRD-CA-06 | docs/PRD.md | Critério de Aceite | Secret antiga válida por 24h após rotação | TRANSCRICAO | [09:21] Sofia |
| PRD-CA-07 | docs/PRD.md | Critério de Aceite | Retry 1m/5m/30m/2h/12h e depois DLQ | TRANSCRICAO | [09:17] Larissa |
| PRD-CA-08 | docs/PRD.md | Critério de Aceite | Replay por ADMIN, OPERATOR recebe 403 | TRANSCRICAO | [09:36] Larissa |
| PRD-CA-09 | docs/PRD.md | Critério de Aceite | Reenvios mantêm o mesmo X-Event-Id | TRANSCRICAO | [09:25] Diego |
| PRD-CA-10 | docs/PRD.md | Critério de Aceite | Histórico mostra últimas 100 tentativas | TRANSCRICAO | [09:34] Marcos |
| PRD-CA-11 | docs/PRD.md | Critério de Aceite | API e worker independentes | TRANSCRICAO | [09:11] Diego |
| PRD-CA-12 | docs/PRD.md | Critério de Aceite | Falha no registro do evento impede mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-TEST-01 | docs/PRD.md | Restrição | Testes de integração seguem padrão Vitest + Supertest existente | CODIGO | tests/orders.test.ts |
| PRD-TEST-02 | docs/PRD.md | Critério de Aceite | Revisão de segurança como etapa de validação | TRANSCRICAO | [09:46] Sofia |
| PRD-TEST-03 | docs/PRD.md | Critério de Aceite | Pós-lançamento: medir impacto antes de decidir e-mail/rate limit | TRANSCRICAO | [09:37] Larissa |
| RFC-META-01 | docs/RFC.md | Contexto | Revisores = participantes da reunião; revisão com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |
| RFC-CTX-01 | docs/RFC.md | Contexto | changeStatus é transação pesada (orders, history, estoque) | TRANSCRICAO | [09:04] Bruno |
| RFC-CTX-02 | docs/RFC.md | Contexto | changeStatus roda em prisma.$transaction | CODIGO | src/modules/orders/order.service.ts |
| RFC-CTX-03 | docs/RFC.md | Restrição | Escopo só outbound | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-04 | docs/RFC.md | Restrição | Prazo alvo fim de novembro | TRANSCRICAO | [09:45] Marcos |
| RFC-PROP-01 | docs/RFC.md | Decisão | CRUD de configuração com filtro de status e secret | TRANSCRICAO | [09:31] Marcos |
| RFC-PROP-02 | docs/RFC.md | Decisão | publishWebhookEvent(tx, …) grava snapshot na outbox dentro da transação | TRANSCRICAO | [09:41] Bruno |
| RFC-PROP-03 | docs/RFC.md | Decisão | Worker separado, polling 2s, timeout 10s | TRANSCRICAO | [09:09] Diego |
| RFC-PROP-04 | docs/RFC.md | Decisão | Retry + DLQ + replay ADMIN | TRANSCRICAO | [09:17] Larissa |
| RFC-PROP-05 | docs/RFC.md | Decisão | HMAC, secret por endpoint, TLS, 64 KB | TRANSCRICAO | [09:22] Sofia |
| RFC-PROP-06 | docs/RFC.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| RFC-PROP-07 | docs/RFC.md | Decisão | Módulo src/modules/webhooks com padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Disparo síncrono no changeStatus — trava transação, rollback indevido | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams — infra extra, overengineering | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Trigger MySQL — sem LISTEN/NOTIFY, solução improvisada | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Retry indefinido / 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa Descartada | Exactly-once — exige coordenação bilateral | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa Descartada | Secret global — vazamento compromete todos | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-07 | docs/RFC.md | Alternativa Descartada | Renderizar payload no envio — estado inconsistente | TRANSCRICAO | [09:52] Larissa |
| RFC-Q-01 | docs/RFC.md | Questão em Aberto | Rate limiting de saída | TRANSCRICAO | [09:39] Diego |
| RFC-Q-02 | docs/RFC.md | Questão em Aberto | E-mail em falhas repetidas (próxima fase) | TRANSCRICAO | [09:37] Marcos |
| RFC-Q-03 | docs/RFC.md | Questão em Aberto | Escala horizontal do worker (particionar por order_id / lock) | TRANSCRICAO | [09:13] Diego |
| RFC-Q-04 | docs/RFC.md | Questão em Aberto | Endurecer autorização do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-Q-05 | docs/RFC.md | Questão em Aberto | Arquivamento de entregues | TRANSCRICAO | [09:08] Diego |
| RFC-Q-06 | docs/RFC.md | Questão em Aberto | "5 tentativas" total vs retentativas (intervalos somam ~15h) | TRANSCRICAO | [09:48] Larissa |
| RFC-Q-07 | docs/RFC.md | Questão em Aberto | Mecânica da secret antiga "válida em paralelo" no outbound | TRANSCRICAO | [09:21] Sofia |
| RFC-Q-08 | docs/RFC.md | Questão em Aberto | Backoff pode inverter ordem por pedido mesmo com single-worker | TRANSCRICAO | [09:12] Diego |
| RFC-Q-09 | docs/RFC.md | Questão em Aberto | Sem endpoint de listagem da DLQ | TRANSCRICAO | [09:18] Diego |
| RFC-IMP-01 | docs/RFC.md | Trade-off | Caminho crítico do changeStatus ganha leitura de webhooks + insert na outbox | TRANSCRICAO | [09:40] Bruno |
| RFC-IMP-02 | docs/RFC.md | Trade-off | Segundo processo para operar | TRANSCRICAO | [09:11] Diego |
| RFC-IMP-03 | docs/RFC.md | Contexto | Nenhuma tabela existente alterada; novas tabelas no schema Prisma | CODIGO | prisma/schema.prisma |
| RFC-IMP-04 | docs/RFC.md | Risco | Prazo apertado, revisão de segurança reservada | TRANSCRICAO | [09:46] Sofia |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Transactional Outbox no MySQL, na mesma transação do changeStatus | TRANSCRICAO | [09:08] Larissa |
| ADR-001-CTX | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | changeStatus em $transaction com orders, history e estoque | CODIGO | src/modules/orders/order.service.ts |
| ADR-001-CTX2 | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | Regras de débito/reposição de estoque | CODIGO | src/modules/orders/order.status.ts |
| ADR-001-IDX | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Índices em status e created_at; lote pequeno | TRANSCRICAO | [09:08] Diego |
| ADR-001-FILT | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Filtro de eventos aplicado na inserção | TRANSCRICAO | [09:34] Bruno |
| ADR-001-ALT1 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Disparo síncrono | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT2 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Redis Streams | TRANSCRICAO | [09:07] Larissa |
| ADR-001-ALT3 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Publicar fora da transação perde garantia | TRANSCRICAO | [09:41] Diego |
| ADR-001-INFRA | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | Infra atual é só MySQL 8 | CODIGO | docker-compose.yml |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em processo separado, polling de 2s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-SEP | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker não pode rodar no processo da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-ENTRY | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Entry-point src/worker.ts e script npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-002-ENTRY-CODE | docs/adrs/ADR-002-worker-separado-em-polling.md | Integração | Modelo de entry-point e shutdown | CODIGO | src/server.ts |
| ADR-002-PRISMA | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | PrismaClient próprio, mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-002-PRISMA-CODE | docs/adrs/ADR-002-worker-separado-em-polling.md | Integração | createPrismaClient() reutilizado pelo worker | CODIGO | src/config/database.ts |
| ADR-002-ORDER | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Single-worker, ordem por created_at / order_id | TRANSCRICAO | [09:12] Diego |
| ADR-002-ALT1 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Trigger de banco para acordar worker | TRANSCRICAO | [09:09] Bruno |
| ADR-002-ALT2 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Múltiplos workers — perde ordenação | TRANSCRICAO | [09:13] Diego |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | 5 retentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-DLQ | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | DLQ em tabela separada webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| ADR-003-ALT1 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Alternativa Descartada | Retry indefinido | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT2 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Alternativa Descartada | 3 tentativas | TRANSCRICAO | [09:16] Bruno |
| ADR-003-ALT3 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Alternativa Descartada | Marcar failed na própria outbox | TRANSCRICAO | [09:17] Larissa |
| ADR-003-TO | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Trade-off | Evento pode chegar ~15h depois; aceitável para o PM | TRANSCRICAO | [09:17] Marcos |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-ALG | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Algoritmo SHA-256 (padrão de mercado) | TRANSCRICAO | [09:20] Sofia |
| ADR-004-CFG | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Config guarda url, secret, customer_id, ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-004-TLS | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | https obrigatório via schema Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-004-ALT1 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa Descartada | Secret global | TRANSCRICAO | [09:21] Sofia |
| ADR-004-REDACT | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Integração | Adicionar *.secret ao redact do Pino | CODIGO | src/shared/logger/index.ts |
| ADR-004-REV | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Dependência | Revisão de segurança de HMAC e geração de secret | TRANSCRICAO | [09:46] Sofia |
| ADR-004-GRACE | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Escolha de Implementação | Proposta: duas assinaturas em X-Signature durante o grace (a validar) | TRANSCRICAO | [09:21] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id (UUID gerado na entrada da outbox) | TRANSCRICAO | [09:26] Larissa |
| ADR-005-UUID | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | event_id UUID único por evento, cliente dedupica | TRANSCRICAO | [09:25] Diego |
| ADR-005-ALT1 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Alternativa Descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| ADR-005-TO | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de dedup transferida ao cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-005-DOC | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Dependência | Documentar no portal | TRANSCRICAO | [09:26] Marcos |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, Zod, códigos | TRANSCRICAO | [09:30] Larissa |
| ADR-006-MOD | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Módulo src/modules/webhooks com controller/service/repository/routes/schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-MOD-CODE | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Contexto | Padrão de módulo existente | CODIGO | src/modules/customers/customer.routes.ts |
| ADR-006-ERR | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Códigos com prefixo WEBHOOK_ | TRANSCRICAO | [09:29] Larissa |
| ADR-006-ERR-CODE | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Contexto | AppError com statusCode/errorCode/details | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-ERR-CODE2 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | NotFoundError fixa NOT_FOUND → WebhookNotFoundError próprio | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-MW | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Error middleware trata AppError sem mudança | TRANSCRICAO | [09:29] Bruno |
| ADR-006-MW-CODE | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Contexto | Serialização de AppError/Zod/Prisma | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-VAL-CODE | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | validate() converte ZodError em VALIDATION_ERROR | CODIGO | src/middlewares/validate.middleware.ts |
| ADR-006-AUTH | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso de requireRole no replay | TRANSCRICAO | [09:36] Larissa |
| ADR-006-AUTH-CODE | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Contexto | authenticate / requireRole | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-006-WIRE-CODE | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Contexto | Composição de dependências em buildControllers | CODIGO | src/app.ts |
| ADR-006-UUID | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | PKs UUID, "tudo é uuid" | TRANSCRICAO | [09:51] Larissa |
| ADR-006-ALT1 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Alternativa Descartada | Injetar repository no OrderService (preferida função pura com tx) | TRANSCRICAO | [09:41] Diego |
| ADR-006-ALT2 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Alternativa Descartada | ID auto-incremental na outbox | TRANSCRICAO | [09:51] Diego |
| ADR-006-TO | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Trade-off | CRUD aberto a qualquer role; endurecer depois | TRANSCRICAO | [09:37] Sofia |
| ADR-007 | docs/adrs/ADR-007-payload-snapshot-na-insercao.md | Decisão | Payload renderizado (snapshot) na inserção | TRANSCRICAO | [09:52] Bruno |
| ADR-007-FMT | docs/adrs/ADR-007-payload-snapshot-na-insercao.md | Decisão | Formato enxuto sem items | TRANSCRICAO | [09:43] Diego |
| ADR-007-64K | docs/adrs/ADR-007-payload-snapshot-na-insercao.md | Restrição | Limite 64 KB, erro em vez de truncar | TRANSCRICAO | [09:24] Diego |
| ADR-007-ALT1 | docs/adrs/ADR-007-payload-snapshot-na-insercao.md | Alternativa Descartada | Guardar só order_id e renderizar no envio | TRANSCRICAO | [09:51] Bruno |
| ADR-007-ALT2 | docs/adrs/ADR-007-payload-snapshot-na-insercao.md | Alternativa Descartada | Truncar payload grande | TRANSCRICAO | [09:23] Sofia |
| ADR-007-DEL | docs/adrs/ADR-007-payload-snapshot-na-insercao.md | Contexto | OrderService.delete remove pedidos PENDING/CANCELLED (snapshot sobrevive) | CODIGO | src/modules/orders/order.service.ts |
| FDD-CTX-01 | docs/FDD.md | Contexto | changeStatus é o único gancho necessário | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-01 | docs/FDD.md | Métrica | Entrega < 10s com cliente saudável | TRANSCRICAO | [09:09] Diego |
| FDD-OBJ-02 | docs/FDD.md | Requisito Não Funcional | Zero divergência status × evento | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-03 | docs/FDD.md | Requisito Não Funcional | Zero impacto de clientes lentos na API | TRANSCRICAO | [09:04] Bruno |
| FDD-OBJ-04 | docs/FDD.md | Requisito Não Funcional | At-least-once com X-Event-Id estável | TRANSCRICAO | [09:24] Diego |
| FDD-OBJ-05 | docs/FDD.md | Requisito Não Funcional | HMAC-SHA256 com secret por endpoint | TRANSCRICAO | [09:22] Sofia |
| FDD-OBJ-06 | docs/FDD.md | Restrição | Nenhuma dependência npm nova; Node ≥ 20 traz fetch/crypto | CODIGO | package.json |
| FDD-EXC-01 | docs/FDD.md | Fora de Escopo | Inbound | TRANSCRICAO | [09:03] Sofia |
| FDD-EXC-02 | docs/FDD.md | Fora de Escopo | E-mail em falhas | TRANSCRICAO | [09:37] Larissa |
| FDD-EXC-03 | docs/FDD.md | Fora de Escopo | Rate limiting | TRANSCRICAO | [09:39] Larissa |
| FDD-EXC-04 | docs/FDD.md | Fora de Escopo | Dashboard | TRANSCRICAO | [09:40] Larissa |
| FDD-EXC-05 | docs/FDD.md | Fora de Escopo | Arquivamento de entregues | TRANSCRICAO | [09:08] Diego |
| FDD-EXC-06 | docs/FDD.md | Fora de Escopo | Múltiplos workers / ordem global | TRANSCRICAO | [09:13] Larissa |
| FDD-EXC-07 | docs/FDD.md | Fora de Escopo | Items no payload | TRANSCRICAO | [09:43] Diego |
| FDD-EXC-08 | docs/FDD.md | Fora de Escopo | Criação de pedido não emite evento (OrderService.create inalterado) | CODIGO | src/modules/orders/order.service.ts |
| FDD-FLX-01 | docs/FDD.md | Decisão | Fluxo de publicação na outbox dentro do changeStatus | TRANSCRICAO | [09:40] Bruno |
| FDD-FLX-01-R1 | docs/FDD.md | Restrição | Falha no insert da outbox → rollback total | TRANSCRICAO | [09:41] Diego |
| FDD-FLX-01-R2 | docs/FDD.md | Decisão | Só webhooks ativos que assinam o to_status geram linha | TRANSCRICAO | [09:34] Bruno |
| FDD-FLX-01-R3 | docs/FDD.md | Escolha de Implementação | Fan-out: uma linha por webhook-alvo, mesmo event_id | TRANSCRICAO | [09:25] Diego |
| FDD-FLX-01-R4 | docs/FDD.md | Decisão | Payload snapshot na inserção | TRANSCRICAO | [09:52] Larissa |
| FDD-FLX-01-TX | docs/FDD.md | Integração | tx é Prisma.TransactionClient (TxClient) já tipado | CODIGO | src/modules/orders/order.service.ts |
| FDD-FLX-02 | docs/FDD.md | Decisão | Worker: claim de lote PENDING mais antigo, processamento sequencial | TRANSCRICAO | [09:09] Diego |
| FDD-FLX-02-R6 | docs/FDD.md | Escolha de Implementação | recoverStuck() de PROCESSING na partida (seguro por ser single-worker) | TRANSCRICAO | [09:12] Diego |
| FDD-FLX-02-R7 | docs/FDD.md | Escolha de Implementação | Checagem de 64 KB no worker (não na transação, para não bloquear status) | TRANSCRICAO | [09:24] Larissa |
| FDD-FLX-02-R8 | docs/FDD.md | Escolha de Implementação | Webhook inativo no envio → DLQ WEBHOOK_INACTIVE | TRANSCRICAO | [09:21] Bruno |
| FDD-FLX-03 | docs/FDD.md | Decisão | handleFailure com BACKOFF e MAX_ATTEMPTS = 1 + 5 | TRANSCRICAO | [09:17] Diego |
| FDD-FLX-03-FAIL | docs/FDD.md | Decisão | Timeout de 10s conta como falha e vai para retry | TRANSCRICAO | [09:42] Diego |
| FDD-FLX-04 | docs/FDD.md | Decisão | Replay recoloca na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-04-AUD | docs/FDD.md | Requisito Não Funcional | Replay registra autor (log + replayed_by_id) | TRANSCRICAO | [09:36] Sofia |
| FDD-FLX-05 | docs/FDD.md | Decisão | Rotação: previousSecret válido por 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-FLX-06 | docs/FDD.md | Integração | Shutdown SIGINT/SIGTERM + $disconnect no molde do server.ts | CODIGO | src/server.ts |
| FDD-MOD-01 | docs/FDD.md | Decisão | Tabelas webhook_endpoints/outbox/deliveries/dead_letter em Prisma | CODIGO | prisma/schema.prisma |
| FDD-MOD-02 | docs/FDD.md | Decisão | Estados PENDING/PROCESSING/DELIVERED/FAILED da outbox | TRANSCRICAO | [09:08] Diego |
| FDD-MOD-03 | docs/FDD.md | Decisão | Chaves UUID Char(36) | TRANSCRICAO | [09:51] Larissa |
| FDD-MOD-04 | docs/FDD.md | Decisão | DLQ guarda payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-MOD-05 | docs/FDD.md | Decisão | webhook_endpoints guarda url, secret, customer_id, active | TRANSCRICAO | [09:21] Bruno |
| FDD-CONTRATO-00 | docs/FDD.md | Contrato | Rotas sob /api/v1 com envelope de erro padrão | CODIGO | src/app.ts |
| FDD-CONTRATO-00b | docs/FDD.md | Restrição | customerId no body/query, não no JWT | TRANSCRICAO | [09:32] Larissa |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks → 201 com secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-01b | docs/FDD.md | Restrição | events exclui PENDING (nenhuma transição leva a PENDING) | CODIGO | src/modules/orders/order.status.ts |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /webhooks?customerId paginado, sem secret | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-02b | docs/FDD.md | Integração | Paginação via paginated() | CODIGO | src/shared/http/response.ts |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /webhooks/:id (url, events, active) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /webhooks/:id → 204 | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /webhooks/:id/rotate-secret → nova secret + previousSecretExpiresAt | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries → últimas 100 | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay → 202, ADMIN | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | POST de saída com Content-Type, X-Event-Id, X-Webhook-Id, X-Timestamp, X-Signature | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-08b | docs/FDD.md | Contrato | Corpo do evento order.status_changed | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-08c | docs/FDD.md | Contrato | Formato order_number ORD-000123 | CODIGO | src/modules/orders/order.service.ts |
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND (404) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_CUSTOMER_NOT_FOUND (404) — prefixo para tudo do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL via Zod → VALIDATION_ERROR | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-03b | docs/FDD.md | Restrição | validate() fixa código VALIDATION_ERROR | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_INACTIVE (409 / interno) — estado ativo do cadastro | TRANSCRICAO | [09:21] Bruno |
| FDD-ERR-05 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND (404) | TRANSCRICAO | [09:35] Diego |
| FDD-ERR-06 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED (409) | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-07 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED (interno → DLQ) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-08 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (interno → DLQ) | TRANSCRICAO | [09:24] Larissa |
| FDD-ERR-09 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT (retry) | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-10 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_HTTP_ERROR (retry) | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-11 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_NETWORK_ERROR (retry) — cliente offline | TRANSCRICAO | [09:14] Larissa |
| FDD-ERR-12 | docs/FDD.md | Erro | WEBHOOK_MAX_ATTEMPTS_EXCEEDED (→ DLQ) | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-13 | docs/FDD.md | Erro | UNAUTHORIZED/FORBIDDEN reutilizados | CODIGO | src/shared/errors/http-errors.ts |
| FDD-RES-01 | docs/FDD.md | Decisão | Desacoplamento via outbox transacional | TRANSCRICAO | [09:06] Diego |
| FDD-RES-02 | docs/FDD.md | Decisão | Timeout 10s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-03 | docs/FDD.md | Decisão | Backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| FDD-RES-04 | docs/FDD.md | Decisão | Teto 1 + 5 tentativas (~14,6h) | TRANSCRICAO | [09:17] Diego |
| FDD-RES-05 | docs/FDD.md | Decisão | Fallback DLQ + replay manual | TRANSCRICAO | [09:18] Diego |
| FDD-RES-06 | docs/FDD.md | Escolha de Implementação | Falhas não recuperáveis direto para DLQ | TRANSCRICAO | [09:23] Sofia |
| FDD-RES-07 | docs/FDD.md | Decisão | Polling de 2s sem sobreposição | TRANSCRICAO | [09:09] Diego |
| FDD-RES-08 | docs/FDD.md | Escolha de Implementação | WEBHOOK_BATCH_SIZE = 10 ("batch pequeno") | TRANSCRICAO | [09:08] Diego |
| FDD-RES-09 | docs/FDD.md | Escolha de Implementação | Recuperação de PROCESSING órfão (at-least-once) | TRANSCRICAO | [09:24] Diego |
| FDD-RES-10 | docs/FDD.md | Decisão | Isolamento de processo worker ≠ API | TRANSCRICAO | [09:11] Diego |
| FDD-RES-11 | docs/FDD.md | Fora de Escopo | Sem fallback por e-mail nesta fase | TRANSCRICAO | [09:37] Larissa |
| FDD-RES-ENV | docs/FDD.md | Integração | Variáveis WEBHOOK_* com default no envSchema | CODIGO | src/config/env.ts |
| FDD-OBS-01 | docs/FDD.md | Restrição | Observabilidade sem libs novas, apoiada no Pino | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Integração | Logs estruturados via logger/child do Pino | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-03 | docs/FDD.md | Requisito Não Funcional | Redaction de *.secret (histórico de vazamento em log) | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-04 | docs/FDD.md | Métrica | Latência p95 < 10s como métrica principal | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-05 | docs/FDD.md | Métrica | Volume de eventos por webhook como insumo para rate limit | TRANSCRICAO | [09:39] Diego |
| FDD-OBS-06 | docs/FDD.md | Métrica | Log de replay com autor | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-07 | docs/FDD.md | Decisão | event_id como chave de correlação ponta a ponta | TRANSCRICAO | [09:25] Diego |
| FDD-OBS-08 | docs/FDD.md | Integração | Correlação com log http_request (requestId) | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-09 | docs/FDD.md | Decisão | X-Webhook-Id para identificar o cadastro | TRANSCRICAO | [09:45] Diego |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus chama publishWebhookEvent(tx, order, from, to) | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-01b | docs/FDD.md | Decisão | Função pura recebendo tx em vez de injetar repository | TRANSCRICAO | [09:41] Bruno |
| FDD-INT-02 | docs/FDD.md | Integração | order.status.ts como referência de transições (sem alteração) | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Novos models + relação em Customer | CODIGO | prisma/schema.prisma |
| FDD-INT-04 | docs/FDD.md | Integração | Novas classes Webhook*Error no molde de InsufficientStockError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-04b | docs/FDD.md | Integração | Exportação pelo barrel de erros | CODIGO | src/shared/errors/index.ts |
| FDD-INT-05 | docs/FDD.md | Integração | AppError como base | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-06 | docs/FDD.md | Integração | error.middleware sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | authenticate + requireRole('ADMIN'); req.user.id no replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-08 | docs/FDD.md | Integração | validate() com schemas do módulo | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Logger + redactPaths | CODIGO | src/shared/logger/index.ts |
| FDD-INT-10 | docs/FDD.md | Integração | createPrismaClient() no worker | CODIGO | src/config/database.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Novas variáveis de ambiente | CODIGO | src/config/env.ts |
| FDD-INT-12 | docs/FDD.md | Integração | src/worker.ts espelha server.ts | CODIGO | src/server.ts |
| FDD-INT-13 | docs/FDD.md | Integração | buildControllers instancia o módulo | CODIGO | src/app.ts |
| FDD-INT-14 | docs/FDD.md | Integração | Controllers + montagem /webhooks e /admin/webhooks | CODIGO | src/routes/index.ts |
| FDD-INT-15 | docs/FDD.md | Integração | paginated() na listagem | CODIGO | src/shared/http/response.ts |
| FDD-INT-16 | docs/FDD.md | Integração | Scripts worker / worker:dev | CODIGO | package.json |
| FDD-INT-16b | docs/FDD.md | Decisão | Script npm run worker | TRANSCRICAO | [09:11] Larissa |
| FDD-INT-17 | docs/FDD.md | Integração | Limpeza das tabelas webhook_* no beforeEach | CODIGO | tests/setup.ts |
| FDD-INT-17b | docs/FDD.md | Integração | Factory createTestWebhook() | CODIGO | tests/helpers/factories.ts |
| FDD-INT-18 | docs/FDD.md | Decisão | Arquivos novos webhook.worker.ts / processor | TRANSCRICAO | [09:28] Bruno |
| FDD-DEP-01 | docs/FDD.md | Dependência | MySQL 8.0 com colunas JSON | CODIGO | docker-compose.yml |
| FDD-DEP-02 | docs/FDD.md | Dependência | Portal de desenvolvedor (Marcos) | TRANSCRICAO | [09:40] Marcos |
| FDD-DEP-03 | docs/FDD.md | Dependência | Revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
| FDD-RISK-01 | docs/FDD.md | Risco | Worker parado → backlog | TRANSCRICAO | [09:11] Diego |
| FDD-RISK-02 | docs/FDD.md | Risco | Cliente lento atrasa outros (processamento sequencial) | TRANSCRICAO | [09:42] Diego |
| FDD-RISK-03 | docs/FDD.md | Risco | Fora de ordem com retry | TRANSCRICAO | [09:13] Larissa |
| FDD-RISK-04 | docs/FDD.md | Risco | Secret vazada em log | TRANSCRICAO | [09:22] Diego |
| FDD-RISK-05 | docs/FDD.md | Risco | Falha no insert bloqueia status (intencional) | TRANSCRICAO | [09:40] Bruno |
| FDD-RISK-06 | docs/FDD.md | Risco | Crescimento de tabelas sem arquivamento | TRANSCRICAO | [09:08] Diego |
| FDD-RISK-07 | docs/FDD.md | Risco | Cliente não deduplica | TRANSCRICAO | [09:26] Marcos |
| FDD-RISK-08 | docs/FDD.md | Risco | HMAC/geração de secret mal implementados | TRANSCRICAO | [09:46] Sofia |
| FDD-CA-01 | docs/FDD.md | Critério de Aceite | Uma linha PENDING por webhook-alvo na mesma transação | TRANSCRICAO | [09:40] Bruno |
| FDD-CA-02 | docs/FDD.md | Critério de Aceite | Exceção no publish não altera status/estoque | TRANSCRICAO | [09:41] Diego |
| FDD-CA-03 | docs/FDD.md | Critério de Aceite | Sem assinante, nenhuma linha | TRANSCRICAO | [09:34] Bruno |
| FDD-CA-04 | docs/FDD.md | Critério de Aceite | Headers de saída presentes | TRANSCRICAO | [09:44] Diego |
| FDD-CA-05 | docs/FDD.md | Critério de Aceite | X-Signature verificável | TRANSCRICAO | [09:20] Sofia |
| FDD-CA-06 | docs/FDD.md | Critério de Aceite | Agenda de retry e DLQ | TRANSCRICAO | [09:17] Larissa |
| FDD-CA-07 | docs/FDD.md | Critério de Aceite | > 64 KB direto para DLQ | TRANSCRICAO | [09:24] Larissa |
| FDD-CA-08 | docs/FDD.md | Critério de Aceite | Replay ADMIN, mesmo event_id, autor registrado | TRANSCRICAO | [09:36] Sofia |
| FDD-CA-09 | docs/FDD.md | Critério de Aceite | http → 400; secret só em create/rotate | TRANSCRICAO | [09:23] Sofia |
| FDD-CA-10 | docs/FDD.md | Critério de Aceite | Duas assinaturas no grace de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CA-11 | docs/FDD.md | Critério de Aceite | deliveries ≤ 100, desc | TRANSCRICAO | [09:34] Marcos |
| FDD-CA-12 | docs/FDD.md | Critério de Aceite | Códigos WEBHOOK_ via error middleware | TRANSCRICAO | [09:29] Bruno |
| FDD-CA-13 | docs/FDD.md | Critério de Aceite | Nenhum log com secret | TRANSCRICAO | [09:22] Diego |
| FDD-CA-14 | docs/FDD.md | Critério de Aceite | npm run worker independente da API | TRANSCRICAO | [09:11] Diego |

# ADR-006 — Reuso máximo dos padrões existentes do projeto (módulo, erros, logger, middlewares)

- **Status:** Aceito
- **Data:** reunião técnica de kickoff (quinta-feira, 09:00)
- **Decisores:** Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)

## Contexto

A codebase do OMS tem padrões consolidados [09:27 Bruno]:

| Padrão | Onde está no código |
|---|---|
| Módulo por domínio com `controller`, `service`, `repository`, `routes`, `schemas` | `src/modules/orders/`, `src/modules/customers/`, `src/modules/products/`, `src/modules/users/` |
| Composição manual de dependências | `buildControllers()` em `src/app.ts`; tipo `Controllers` e `buildApiRouter()` em `src/routes/index.ts` |
| Hierarquia de erros com `statusCode` + `errorCode` | `AppError` em `src/shared/errors/app-error.ts`; subclasses em `src/shared/errors/http-errors.ts` (ex.: `InvalidStatusTransitionError` → `INVALID_STATUS_TRANSITION`, `InsufficientStockError` → `INSUFFICIENT_STOCK`) |
| Tratamento centralizado de erros (AppError, ZodError, Prisma P2002/P2025) | `src/middlewares/error.middleware.ts` |
| Validação por schema Zod em body/query/params | `validate()` em `src/middlewares/validate.middleware.ts` |
| Autenticação JWT e autorização por papel | `authenticate` e `requireRole()` em `src/middlewares/auth.middleware.ts` |
| Logger estruturado Pino com redaction | `src/shared/logger/index.ts` |
| Paginação padronizada | `paginated()` em `src/shared/http/response.ts` |
| Chaves UUID em todas as tabelas | `prisma/schema.prisma` (`@id @default(uuid()) @db.Char(36)`) |

O time é pequeno e o prazo é curto: três sprints, com meta no fim de novembro [09:45 Marcos, 09:47 Larissa].

## Decisão

**Reuso máximo do que já existe** [09:30 Larissa]. O webhook é "um módulo igual aos outros":

1. **Módulo `src/modules/webhooks/`** com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`, mais `webhook.worker.ts` (processamento) e a função `publishWebhookEvent` [09:27 Bruno, 09:28 Bruno].
2. **Erros:** as novas classes estendem `AppError`, e todos os códigos do módulo usam o prefixo **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) [09:28 Bruno, 09:29 Larissa]. Como `NotFoundError` fixa o código `NOT_FOUND` no construtor (`http-errors.ts`), o `WEBHOOK_NOT_FOUND` exige uma classe própria sobre `AppError`.
3. **Error middleware sem alteração:** `error.middleware.ts` já serializa qualquer `AppError` como `{ error: { code, message, details } }` [09:29 Bruno].
4. **Logger:** Pino em `src/shared/logger/index.ts`. Nenhuma biblioteca nova de logging [09:29 Bruno].
5. **Validação:** schemas Zod + `validate()`, inclusive a regra de URL `https` [09:23 Sofia].
6. **Autorização:** `authenticate` no CRUD e `requireRole('ADMIN')` no replay da DLQ [09:36 Larissa].
7. **Worker:** nova entry-point `src/worker.ts` no molde de `src/server.ts`, com `PrismaClient` próprio via `createPrismaClient()` [09:11 Larissa, 09:30 Bruno].
8. **Identificadores UUID** também nas tabelas de webhook [09:51 Larissa].

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Serviço/repositório separado para webhooks** (microserviço) | Contraria "mesmo banco, mesma stack" [09:11 Diego] e aumenta a carga operacional de um time pequeno [09:07 Diego]. |
| **Injetar um `WebhookRepository` inteiro no `OrderService`** | Acoplamento maior que o necessário. Foi preferida uma função pura `publishWebhookEvent(tx, …)` que recebe o client da transação [09:41 Bruno, 09:41 Diego]. |
| **Códigos de erro genéricos** (`NOT_FOUND`, `BAD_REQUEST`) para o módulo | O cliente não consegue distinguir erros de webhook de erros de outros domínios. O time optou pelo prefixo `WEBHOOK_` [09:29 Larissa]. |
| **Chave primária auto-incremental na outbox** | Diverge do padrão do projeto, em que "tudo é uuid" [09:51 Diego, 09:51 Larissa]. |

## Consequências

**Positivas**
- Curva de aprendizado zero para o time. Code review segue os mesmos critérios dos outros módulos.
- Menos código novo: erros, logs, validação e autenticação já testados.
- Respostas de erro com o mesmo envelope do resto da API, sem surpresa para integradores.

**Negativas / trade-offs**
- **O validate middleware sempre converte `ZodError` em `VALIDATION_ERROR`** (`validate.middleware.ts`). Uma regra validada no Zod (como `https`) **não produz** `WEBHOOK_INVALID_URL` como `error.code`. O [FDD](../FDD.md#7-matriz-de-erros) resolve colocando o código `WEBHOOK_*` na mensagem do `details`, e a divergência fica registrada no RFC.
- O worker **não** passa pelo `error.middleware.ts`, que é exclusivo do Express, e precisa tratar e logar os próprios erros.
- Com o padrão CRUD atual, qualquer usuário autenticado (ADMIN ou OPERATOR) gerencia webhooks de qualquer cliente. É aceito por ora, com endurecimento futuro [09:37 Sofia].

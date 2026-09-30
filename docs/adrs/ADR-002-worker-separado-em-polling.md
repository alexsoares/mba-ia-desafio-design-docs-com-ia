# ADR-002 — Worker em processo separado, consumindo a outbox por polling de 2s

- **Status:** Aceito
- **Data:** reunião técnica de kickoff (quinta-feira, 09:00)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)

## Contexto

Com a outbox definida no [ADR-001](ADR-001-outbox-no-mysql.md), falta decidir **como** e **onde** os eventos são consumidos.

- O requisito de negócio é "abaixo de 10 segundos" [09:02 Marcos].
- O MySQL não tem mecanismo nativo de notificação a processos externos, como o `LISTEN/NOTIFY` do PostgreSQL. Triggers só executam SQL [09:09 Diego].
- Hoje a aplicação tem uma única entry-point, `src/server.ts`, que sobe o Express e trata `SIGINT`/`SIGTERM` com `prisma.$disconnect()`.
- O `PrismaClient` é criado por processo em `src/config/database.ts` (`createPrismaClient()`).

## Decisão

1. **Consumo por polling em loop:** a cada **2 segundos** o worker busca os eventos pendentes mais antigos, em **lote pequeno**, processa e marca o resultado [09:09 Diego, 09:10 Larissa].
2. **Processo separado da API:** o worker **não** roda dentro da instância HTTP. Se a API reiniciar, o worker continua, e vice-versa [09:11 Diego].
3. **Nova entry-point `src/worker.ts`**, espelhando `src/server.ts`, com o script `npm run worker` [09:11 Larissa]. A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.worker.ts` (ou `webhook.processor.ts`) [09:28 Bruno].
4. **Mesmo banco e mesma stack, `PrismaClient` próprio:** mesma `DATABASE_URL`, mas uma instância nova, porque é outro processo Node [09:11 Bruno, 09:30 Bruno]. Reusa-se `createPrismaClient()` de `src/config/database.ts`.
5. **Instância única (single-worker)** nesta fase. Os eventos são processados em ordem de `created_at`, o que dá ordenação implícita por `order_id` [09:12 Diego].

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Trigger no MySQL para acordar o worker** | O MySQL não notifica processos externos. Seria preciso improvisar (escrever em arquivo, chamar um endpoint), e o próprio time achou a solução "esquisita" [09:09 Bruno, 09:09 Diego]. |
| **Worker dentro do processo da API** (ex.: `setInterval` em `server.ts`) | O ciclo de vida fica acoplado: um restart ou deploy da API derruba o worker [09:11 Diego]. |
| **Múltiplos workers em paralelo** | Perde-se a ordenação por pedido sem um mecanismo adicional (particionar por `order_id` ou lock pessimista). Ficou como "problema do futuro" [09:12 Diego, 09:13 Diego]. |

## Consequências

**Positivas**
- 2s de polling cabem com folga no requisito de < 10s [09:09 Diego, 09:10 Marcos].
- Implementação simples, sem dependência nova, testável com o mesmo Vitest e o mesmo banco.
- Ciclo de vida e deploy independentes da API.

**Negativas / trade-offs**
- **Latência mínima de até 2s** por evento, aceita formalmente [09:10 Larissa].
- Consultas periódicas ao banco mesmo sem eventos. O custo é baixo graças aos índices de `status`/`created_at` [09:08 Diego].
- **Limitação conhecida:** não há garantia de ordenação global, apenas por `order_id`, e só enquanto houver um único worker [09:13 Larissa]. Os clientes nunca pediram ordenação global [09:14 Marcos].
- Operação passa a ter **dois processos** para monitorar e fazer deploy (`npm start` e `npm run worker`).
- Um worker único é ponto único de atraso: se ele parar, os eventos se acumulam na outbox. Nada se perde, mas a entrega atrasa.

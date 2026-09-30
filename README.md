# Da Reunião ao Documento: Design Docs Gerados por IA

Pacote de design docs da feature **Sistema de Webhooks de Notificação de Pedidos** do OMS (Order Management System), produzido a partir de `TRANSCRICAO.md` e do código deste repositório, com IA como ferramenta principal de produção.

> O enunciado original do desafio está no histórico do repositório base (commit `e7f6311`, arquivo `README.md`).

## Sobre o desafio

Uma empresa com um OMS em produção (Node.js + TypeScript, Express, Prisma/MySQL) decidiu numa reunião de ~55 minutos construir webhooks de saída que avisem clientes B2B quando o status de um pedido muda. A reunião teve tech lead, PM, dois engenheiros e segurança, e a única coisa registrada foi a transcrição literal. A tarefa foi transformar essa conversa, junto com a leitura do código existente, em documentação acionável: PRD, RFC, FDD, ADRs e um tracker que prova de onde veio cada afirmação.

O trabalho principal não foi escrever, e sim **filtrar e reconciliar**. A reunião mistura decisões fechadas, ideias descartadas (Redis, retry indefinido, secret global), itens adiados (e-mail, rate limiting, dashboard) e detalhes que contradizem uns aos outros ou o próprio código. A regra do desafio é que nada pode ser inventado. Por isso toda lacuna precisou virar uma questão em aberto explícita ou uma "escolha de implementação" rotulada e rastreada até a fala que a motivou.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code (CLI) com Claude Opus 5.5** | Ferramenta única de produção. Leu o repositório inteiro (`src/`, `prisma/`, `tests/`, configs) e a transcrição, fez a filtragem dirigida (decidido / descartado / adiado / aberto), redigiu os documentos e executou a revisão |
| **Script de validação em Python** (gerado pela IA, executado localmente) | Rede de segurança contra alucinação. Confere cada `[hh:mm] Falante` do tracker e dos documentos contra a transcrição, confere se todo caminho de código citado existe, calcula as porcentagens de fonte do tracker e verifica se todo ID dos documentos tem linha no tracker e se os links relativos resolvem |

## Workflow adotado

1. **Contextualização:** leitura integral de `TRANSCRICAO.md` e dos arquivos de código relevantes: `order.service.ts`, `order.status.ts`, `schema.prisma`, erros, middlewares, logger, `app.ts`, `server.ts`, `routes/index.ts`, `env.ts`, testes.
2. **Filtragem dirigida da transcrição** em quatro baldes: decisões fechadas, requisitos, descartados/adiados e detalhes secundários. Em paralelo, **cruzamento com o código**, para achar onde a fala da reunião não encaixa na implementação real (ver *Iterações*).
3. **ADRs primeiro** (7 no total: as 6 decisões principais + snapshot do payload). Eles viraram o esqueleto do resto.
4. **RFC** em cima dos ADRs, curto (~1.500 palavras), com alternativas descartadas e 9 questões em aberto.
5. **FDD**, o documento mais profundo: fluxos, modelo Prisma, 8 contratos, matriz de erros `WEBHOOK_*`, resiliência, observabilidade e a integração arquivo por arquivo (17 arquivos reais).
6. **PRD** por último entre os grandes. Com o resto pronto, virou consolidação de negócio.
7. **Tracker**: varredura de todos os documentos, com IDs estáveis (`PRD-FR-07`, `FDD-CONTRATO-05`, …) usados dentro dos próprios documentos.
8. **Validação automática + revisão semântica**, com correções.
9. **README** (este arquivo).

Organização da interação com a IA:

- uma única sessão com contexto acumulado (código + transcrição + documentos já escritos);
- cada documento gerado com o anterior como insumo;
- cada citação no formato `[hh:mm Falante]` desde o primeiro rascunho, para que o tracker fosse uma extração e não uma reconstrução posterior.

## Prompts customizados

Os prompts abaixo registram as instruções dirigidas usadas em cada passada da sessão, em vez de pedidos genéricos.

**Prompt 1: filtragem dirigida da transcrição.** Evita o "gere um PRD a partir disso", que mistura descartado com decidido.

```text
Leia TRANSCRICAO.md inteira. NÃO resuma. Classifique cada fala relevante em exatamente um balde:
  (A) DECISÃO FECHADA — alguém fechou explicitamente ("decidido", "anotado", "tá decidido",
      concordância sem objeção). Cite quem fechou.
  (B) REQUISITO FUNCIONAL/NÃO FUNCIONAL — pedido do PM ou restrição técnica/segurança.
  (C) DESCARTADO — proposta rejeitada. Registre o trade-off que motivou o descarte.
  (D) ADIADO / EM ABERTO — "fica pra próxima fase", "observar", "problema do futuro".
  (E) DETALHE SECUNDÁRIO — payload, headers, timeouts, nomes de arquivo.
Para cada item devolva: balde | resumo em 1 linha | [hh:mm] Falante da fala DECISIVA.
Regras: não promova (C) ou (D) a requisito. Se duas falas se contradizem, liste as duas
e marque CONFLITO em vez de escolher. Se um número aparece sem ser confirmado, marque
"proposto por X, confirmado por Y" ou "não confirmado".
```

**Prompt 2: cruzamento transcrição × código.** Foi o que encontrou os conflitos mais importantes.

```text
Para cada decisão do balde (A) e (E), verifique contra o código real:
  - src/modules/orders/order.service.ts (changeStatus, $transaction, TxClient)
  - src/modules/orders/order.status.ts (tabela de transições)
  - src/shared/errors/*.ts (AppError, construtores e códigos fixos)
  - src/middlewares/{validate,error,auth}.middleware.ts
  - src/shared/logger/index.ts (redactPaths), src/config/*.ts, prisma/schema.prisma
Responda: (1) a decisão é implementável como dita, sem mudar o que não foi acordado?
(2) se não, qual arquivo/linha impede e qual a menor adaptação que respeita a decisão?
(3) algum arquivo citado na reunião não existe? Não invente arquivos: só cite caminhos
que você abriu. Separe o que é "novo arquivo proposto" do que é "arquivo existente".
```

**Prompt 3: geração do FDD com rótulo de origem obrigatório.**

```text
Escreva docs/FDD.md com as seções obrigatórias do enunciado + "Integração com o sistema
existente". Todo valor numérico, nome de header, código de erro e endpoint precisa de
[hh:mm Falante] ou de um caminho de arquivo. Quando a reunião NÃO fixou algo que o FDD
precisa para ser acionável (ex.: tamanho do lote, comportamento com webhook inativo),
decida, mas marque "(escolha de implementação)" e aponte a fala que motivou a escolha.
Não repita o racional das alternativas (isso está nos ADRs/RFC); aqui é "como construir".
Contratos: request e response de exemplo, headers e tabela de status codes por endpoint.
```

**Prompt 4: revisão adversarial.**

```text
Atue como revisor hostil do pacote docs/. Procure: (1) item descartado ou adiado que
aparece como requisito; (2) número divergente entre documentos (tentativas, timeouts,
limites); (3) nome de campo diferente para a mesma coisa em contratos diferentes;
(4) critério de aceite que depende de algo ainda em aberto sem dizer isso;
(5) nível de detalhe de FDD vazando para o RFC. Liste achados com arquivo:linha.
```

## Iterações e ajustes

Foram **quatro ciclos principais**: filtragem → geração (ADRs, RFC, FDD, PRD, Tracker) → validação automática → revisão semântica e correção. Os ajustes concretos:

1. **"5 tentativas": total ou retentativas?** A primeira leitura tratava "5 tentativas" como 5 chamadas HTTP. Mas os intervalos 1m/5m/30m/2h/12h são **cinco esperas**, que somam 876 min ≈ 14,6 h, exatamente o "quase 15 horas" de Diego [09:17]. Com 5 chamadas no total, só caberiam 4 esperas (~2,6 h). O FDD adotou **1 envio + 5 retentativas**, e a divergência com o resumo da Larissa [09:48] virou a questão **RFC Q-06** em vez de ser escondida.
2. **Limite de 64 KB × atomicidade.** A decisão diz "erra se passar de 64 KB" [09:24], mas um erro dentro da transação do `changeStatus` desfaria a mudança de status, o que contraria "não pode ter caso de status mudar e evento não sair" [09:40]. Ajuste: a checagem foi para o **worker**, que manda o evento direto para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem retry.
3. **`WEBHOOK_INVALID_URL` não sai do jeito que foi falado.** Sofia pediu a validação de `https` no Zod [09:23], e Bruno citou o código `WEBHOOK_INVALID_URL` [09:28]. Só que `src/middlewares/validate.middleware.ts` converte **todo** `ZodError` em `VALIDATION_ERROR`. O FDD documenta que o código vai em `details[].message`, sem mexer no middleware, e deixa a alternativa registrada.
4. **`NotFoundError` tem código fixo.** Em `src/shared/errors/http-errors.ts`, o construtor de `NotFoundError` sempre usa `NOT_FOUND`. `WEBHOOK_NOT_FOUND` exige uma classe própria sobre `AppError`, o que ficou registrado no ADR-006 e na integração I-04.
5. **`customer_id` vindo do JWT.** Marcos disse "implícito do JWT" [09:31], mas foi corrigido um minuto depois: o JWT é do operador, e o `customer_id` vai no body ou no path [09:32 Larissa]. Uma leitura linear pegaria a fala superada. A regra de CONFLITO do Prompt 1 isolou as duas falas, e os contratos (PRD-FR-02, FDD §6) usam a versão corrigida.
6. **Backoff quebra a ordenação por pedido.** A reunião aceitou "ordem por `order_id` com single-worker" [09:12], mas com retry um evento `PAID` em espera de 1 min é ultrapassado pelo `PROCESSING` seguinte do mesmo pedido. Isso não foi discutido. Virou risco (FDD-RISK-03) e questão **RFC Q-08**, sem inventar mecanismo de bloqueio.
7. **"A secret antiga fica válida por 24h" num webhook de saída.** Quem assina é o OMS, então "válida" só faz sentido se as duas assinaturas forem enviadas. Isso ficou como **proposta** no ADR-004, marcada para a revisão de segurança (Q-07), e o PRD-CA-06 foi anotado como dependente dela.
8. **Filtro de eventos com `PENDING`.** A tabela de transições de `order.status.ts` não tem nenhuma transição **para** `PENDING`, então assinar `PENDING` nunca dispararia nada. O schema passou a rejeitar esse valor.
9. **Revisão semântica (ciclo 4):**
   - nome de campo inconsistente entre contratos (`secretRotationPendingUntil` × `previousSecretExpiresAt`), unificado;
   - `replayedById` de exemplo fora do formato UUID;
   - `docs/adrs/README.md` com convenção de nome (`0001-…`) conflitante com a exigida, reescrito como índice.
10. **Validação automática:** o script confirmou 0 timestamps inválidos, 0 caminhos inexistentes e 100% dos IDs dos documentos presentes no tracker, com 314 linhas (83% TRANSCRICAO, 53 CODIGO). Essa checagem foi repetida depois de cada correção.

## Como navegar a entrega

```
.
├── README.md                 ← este arquivo (processo)
├── TRANSCRICAO.md            ← fonte (inalterada)
└── docs/
    ├── PRD.md                ← 1. por que e o quê (negócio)
    ├── RFC.md                ← 2. proposta técnica, alternativas, questões em aberto
    ├── adrs/
    │   ├── README.md         ← índice dos ADRs
    │   ├── ADR-001-outbox-no-mysql.md
    │   ├── ADR-002-worker-separado-em-polling.md
    │   ├── ADR-003-retry-backoff-exponencial-e-dlq.md
    │   ├── ADR-004-hmac-sha256-com-secret-por-endpoint.md
    │   ├── ADR-005-at-least-once-com-x-event-id.md
    │   ├── ADR-006-reuso-dos-padroes-do-projeto.md
    │   └── ADR-007-payload-snapshot-na-insercao.md
    ├── FDD.md                ← 4. como construir (contratos, fluxos, erros, integração)
    └── TRACKER.md            ← de onde veio cada item
```

**Ordem sugerida de leitura:**

1. [PRD](docs/PRD.md);
2. [RFC](docs/RFC.md);
3. [ADRs](docs/adrs/README.md), na ordem em que o RFC os referencia;
4. [FDD](docs/FDD.md);
5. [TRACKER](docs/TRACKER.md), sempre que quiser conferir a origem de um item. Os IDs `PRD-…`/`FDD-…`/`RFC-…` e as citações `[hh:mm Falante]` nos documentos apontam para ele e para a transcrição.

O código da aplicação (`src/`, `prisma/`, `tests/`, configurações) **não foi alterado**. A entrega é puramente documental.

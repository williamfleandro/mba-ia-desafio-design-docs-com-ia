# ADR-006: Reuso dos padrões arquiteturais existentes no módulo de webhooks

## Status

Aceita — decidida em reunião técnica; implementação ainda não iniciada (ver "Relação com o sistema existente").

## Data

Reunião técnica de quinta-feira, 09:00 (registrada em `TRANSCRICAO.md`; a transcrição não informa a data absoluta, apenas o dia da semana e o horário).

## Decisores

- Bruno (Engenheiro Pleno, time de Pedidos) — propõe estrutura de módulo, códigos de erro, reuso do logger e confirma reuso do Prisma com instância própria por processo
- Diego (Engenheiro Sênior, time de Plataforma) — confirma a estrutura de módulo proposta e levanta a questão sobre instância própria de `PrismaClient` no worker
- Larissa (Tech Lead) — fecha a decisão de reuso máximo do que já existe

## Fontes

- `TRANSCRICAO.md`, `[09:27]-[09:37]`
- `reports/context-analysis.md`, decisões DEC-017, DEC-018, DEC-019, DEC-020, DEC-027, DEC-028, DEC-029, DEC-030; requisito FR-012; pontos de integração CODE-001 a CODE-012
- Código: `src/modules/orders/*`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/shared/logger/index.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/config/database.ts`, `src/app.ts`, `src/routes/index.ts`, `tests/orders.test.ts`, `tests/auth.test.ts`

## Contexto

Ao discutir estrutura de código e padrões para a nova feature, Bruno apontou que o projeto já tem um padrão claro: cada domínio é um módulo em `src/modules/<domínio>` com `controller`, `service`, `repository`, `routes` e `schemas`, além de um padrão consolidado de erros (`AppError` e subclasses com códigos como `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`), logger Pino já usado no projeto inteiro, e um middleware de erro centralizado que já trata `AppError`, `ZodError` e erros do Prisma sem precisar de mudanças (`[09:27]-[09:29] Bruno`).

## Decisão

O módulo de webhooks segue integralmente os padrões já estabelecidos no projeto, sem introduzir infraestrutura nova:

- **Estrutura de módulo**: `src/modules/webhooks/` com `controller`, `service`, `repository`, `routes` e `schemas`, no mesmo formato dos módulos `orders`, `auth`, `users`, `customers` e `products` (`[09:27]-[09:28] Bruno, Diego`).
- **Worker como entry-point separado**: `src/worker.ts` faz o bootstrap do processo; a lógica de processamento fica em um arquivo dentro do módulo (`webhook.worker.ts` ou `webhook.processor.ts`, nome não fechado) (`[09:28] Bruno, Diego` — ver também ADR-002).
- **Erros de domínio**: seguem o padrão `AppError`, com códigos prefixados por `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), da mesma forma que hoje existem `InsufficientStockError` e `InvalidStatusTransitionError` (`[09:28]-[09:29] Bruno, Larissa`).
- **Logger e middleware de erro**: reuso direto do logger Pino já usado em todo o projeto e do `errorMiddleware` central, sem nenhuma alteração — o middleware já trata `AppError`, `ZodError` e erros do Prisma (`[09:29] Bruno`).
- **Banco de dados**: reuso do mesmo MySQL e do padrão de configuração do Prisma; o worker usa uma instância própria de `PrismaClient` por rodar em processo separado (ver ADR-002), mas com a mesma `DATABASE_URL` (`[09:29]-[09:30] Diego, Bruno`).
- **Autenticação e autorização**: reuso direto de `authenticate` e `requireRole` já existentes. Endpoints de CRUD de configuração de webhook aceitam qualquer role autenticada nesta fase (`[09:36]-[09:37] Sofia, Marcos`); o endpoint de replay de DLQ exige role `ADMIN` e log de auditoria de quem executou o replay (`[09:35]-[09:36] Sofia, Larissa` — ver ADR-003). O `customer_id` é passado explicitamente no body/path do endpoint, não derivado do JWT, pois o JWT atual representa um usuário operador do sistema, não o cliente final (`[09:31]-[09:32] Bruno, Larissa, Marcos`).
- **Identificadores**: UUID para os registros do domínio de webhooks, seguindo o padrão já usado em todo o schema Prisma atual (`[09:51] Larissa` — ver também ADR-001).
- **Validação**: novos schemas Zod seguem o mesmo padrão de `order.schemas.ts`, reaproveitando o middleware `validate` já existente.
- **Testes**: seguem o padrão de `tests/orders.test.ts`/`tests/auth.test.ts` — Vitest + Supertest, batendo em uma instância real da aplicação, com asserções diretas no Prisma real, sem mocks.

## Alternativas consideradas

Nenhuma alternativa concorrente foi discutida na reunião para este ponto — a decisão emergiu por consenso direto entre Bruno e Diego ("Faz sentido?" / "Faz", `[09:27]-[09:28]`), sem comparação com abordagens alternativas de estruturação de código, tratamento de erro ou logging.

## Consequências positivas

- Consistência arquitetural entre o novo módulo e o restante do sistema.
- Curva de aprendizado baixa para o time, que já conhece o padrão de módulo, erros e middlewares.
- Nenhuma alteração necessária no middleware de erro central, no logger ou nos middlewares de autenticação/RBAC.

## Consequências negativas

- O módulo webhooks herda quaisquer limitações já presentes nos padrões atuais — por exemplo, a composição manual de dependências sem framework de DI (`buildControllers(prisma)` em `src/app.ts`), que precisa ser estendida manualmente para os novos componentes.
- O worker, apesar de reaproveitar o padrão de criação do Prisma Client, ainda exige a criação explícita de uma nova instância por processo — não é um reuso "gratuito" do singleton da API (ver ADR-002).

## Trade-offs

- **Consistência e baixo custo vs. oportunidade de nova abordagem**: reaproveitar os padrões existentes reduz custo de implementação e revisão, mas significa herdar as restrições arquiteturais atuais (ex.: DI manual) em vez de aproveitar a nova feature para introduzir ferramentas diferentes.

## Relação com o sistema existente

- `src/modules/orders/*` serve de referência estrutural direta para `src/modules/webhooks/*`.
- `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` definem o padrão `AppError` e subclasses (`ConflictError`, `NotFoundError`, `UnprocessableEntityError`, etc.) que as novas classes `WEBHOOK_*` devem estender.
- `src/middlewares/error.middleware.ts` já trata `AppError`, `ZodError` e `Prisma.PrismaClientKnownRequestError` — nenhuma mudança necessária.
- `src/shared/logger/index.ts` fornece o singleton Pino a ser reutilizado pela API e pelo worker.
- `src/middlewares/auth.middleware.ts` fornece `authenticate` e `requireRole(...roles)`, reaproveitados sem alteração.
- `src/middlewares/validate.middleware.ts` fornece o middleware `validate({body, query, params})` para os novos schemas Zod.
- `src/config/database.ts` fornece o padrão de criação do `PrismaClient`, reaproveitado (com instância própria) pelo worker.
- `src/app.ts` (`buildControllers`) e `src/routes/index.ts` (`buildApiRouter`) são os pontos onde os novos `WebhookRepository`/`WebhookService`/`WebhookController` e a rota `/webhooks` (e sub-rota `/admin`, conforme mencionado na reunião) devem ser registrados, seguindo o padrão manual já existente.
- `tests/orders.test.ts` e `tests/auth.test.ts` são a referência de padrão de teste a seguir para o módulo webhooks.
- Nenhum desses artefatos de webhooks existe hoje no repositório — confirmado por busca de `webhook*`/`worker*`, que não retornou nenhum arquivo.

## Rastreabilidade

| Origem | IDs |
| --- | --- |
| Decisões (transcrição) | DEC-017, DEC-018, DEC-019, DEC-020, DEC-027, DEC-028, DEC-029, DEC-030 |
| Requisito relacionado | FR-012 |
| Pontos de integração de código | CODE-001, CODE-002, CODE-003, CODE-004, CODE-005, CODE-006, CODE-007, CODE-008, CODE-009, CODE-010, CODE-011, CODE-012 |
| Timestamps-chave | `[09:27]-[09:37]` |

# ADR-002: Worker separado em processo próprio, lendo a outbox via polling

## Status

Aceita — decidida em reunião técnica; implementação ainda não iniciada (ver "Relação com o sistema existente").

## Data

Reunião técnica de quinta-feira, 09:00 (registrada em `TRANSCRICAO.md`; a transcrição não informa a data absoluta, apenas o dia da semana e o horário).

## Decisores

- Diego (Engenheiro Sênior, time de Plataforma) — propõe polling e processo separado
- Larissa (Tech Lead) — fecha a decisão
- Bruno (Engenheiro Pleno, time de Pedidos) — confirma reuso de banco/Prisma

## Fontes

- `TRANSCRICAO.md`, `[09:08]-[09:14]`, `[09:28]-[09:30]`
- `reports/context-analysis.md`, decisões DEC-003, DEC-004, DEC-005, DEC-006, DEC-018; alternativa ALT-003; NFR-002, NFR-010, NFR-012, NFR-014; limitações LIMIT-001, LIMIT-002, LIMIT-005; risco RISK-006; pontos de integração CODE-010, CODE-011
- Código: `src/server.ts` (padrão de entry point), `src/config/database.ts` (padrão de criação do `PrismaClient`), `package.json` (scripts)

## Contexto

Definido o padrão Outbox (ADR-001), era preciso decidir como um worker consumiria os eventos pendentes. Bruno perguntou se seria possível usar um trigger de banco para reatividade; Diego explicou que o MySQL não tem um mecanismo nativo equivalente ao `NOTIFY/LISTEN` do PostgreSQL, e que triggers só executam SQL — não notificam um processo externo de forma limpa (alternativas de contorno como escrever em arquivo ou bater em um endpoint foram descritas como "esquisitas") (`[09:09] Diego`).

Também foi decidido que o worker precisa rodar como processo separado da API: se rodasse na mesma instância, um reinício da API derrubaria o worker junto (`[09:11] Diego, Larissa`).

## Decisão

- O worker roda como um **processo Node separado** da API, com entry point próprio proposto como `src/worker.ts` e script `npm run worker` (ainda inexistentes no repositório e no `package.json` atual — ver "Relação com o sistema existente").
- O worker lê a outbox via **polling em loop, a cada 2 segundos**: busca os eventos pendentes mais antigos em lote pequeno, processa e marca como entregues (`[09:09]-[09:10] Diego`). Esse intervalo atende com folga o requisito do cliente de latência "tempo real" abaixo de 10 segundos (`[09:02] Marcos`, `[09:10] Marcos`).
- O worker conecta no mesmo banco (mesma `DATABASE_URL`), mas com uma **instância própria de `PrismaClient`**, já que `PrismaClient` é por processo — não pode ser compartilhado com a instância usada pela API (`[09:11]`, `[09:29]-[09:30] Diego, Bruno`).
- A lógica de processamento do worker fica em um arquivo dentro do módulo `src/modules/webhooks/` (nome não fechado entre `webhook.worker.ts` ou `webhook.processor.ts` — `[09:28] Bruno`), separado do bootstrap do processo em `src/worker.ts`.

### Limitação e consequência: ordenação depende do regime single-worker

Ao discutir o comportamento do polling, a equipe observou que a leitura sequencial da outbox por `created_at`, feita por um único worker, preserva a ordem de entrega dos eventos de um mesmo pedido (`[09:12] Diego`). Esse comportamento **não foi tratado como uma decisão arquitetural independente com alternativas avaliadas** — nenhuma opção concorrente para garantir ordenação foi comparada nesta reunião; a equipe apenas registrou o efeito colateral do design já decidido (worker único, processo separado, polling sequencial) e a própria Larissa o classificou explicitamente como limitação a documentar: "Documentamos como limitação conhecida. Não é garantia de ordering global, só por `order_id` e enquanto for single-worker" (`[09:13] Larissa`). Por isso, essa condição é registrada aqui como consequência do ADR-002, e não em um ADR próprio:

- A ordenação de entrega só é garantida por `order_id`, e apenas enquanto o sistema operar em regime de **um único worker**.
- Caso o sistema escale para múltiplos workers em paralelo no futuro, a garantia de ordenação se perde; soluções possíveis citadas foram particionamento por `order_id` ou lock pessimista, mas nenhuma delas foi avaliada ou decidida — ficou marcada como "problema do futuro" (`[09:13] Diego`).
- Os clientes não solicitaram garantia de ordenação global, apenas saber quando cada pedido individual mudou de status (`[09:14] Marcos`), o que torna essa limitação aceitável no escopo atual.

## Alternativas consideradas

- **Trigger de banco de dados para notificação reativa** (`[09:09] Bruno, Diego`): descartado porque o MySQL não possui listener nativo equivalente ao `NOTIFY/LISTEN`; um trigger só executa SQL e não notifica processos externos sem soluções improvisadas consideradas inadequadas (escrever em arquivo, bater em endpoint).
- **Worker executando dentro do mesmo processo da API** (implicitamente descartada, `[09:11] Diego`): rejeitada porque um reinício da API derrubaria o worker junto, interrompendo todas as entregas pendentes (`RISK-006`).

## Consequências positivas

- Resiliência operacional: reinícios da API não afetam a disponibilidade do worker.
- Simplicidade de implementação e operação, sem exigir infraestrutura reativa nova.
- O intervalo de 2 segundos atende com folga ao requisito de latência "tempo real" (<10s) definido pelos clientes.

## Consequências negativas

- Latência mínima de entrega de 2 segundos no pior caso, decorrente do próprio intervalo de polling (`LIMIT-002`).
- Overhead de leitura constante na tabela de outbox, mitigado por índices em `status`/`created_at` e leitura em lote pequeno (ADR-001).
- Ordenação de entrega garantida apenas por `order_id` e apenas em regime single-worker; não há garantia de ordenação global, e essa garantia se perde caso o sistema escale para múltiplos workers no futuro (`LIMIT-001`, `LIMIT-005`).

## Trade-offs

- **Reatividade vs. simplicidade operacional**: polling é mais simples de implementar e operar do que um mecanismo reativo, ao custo de uma latência mínima artificial de 2s.
- **Disponibilidade vs. topologia de processo único**: rodar como processo separado aumenta a resiliência a reinícios da API, mas introduz mais um processo para monitorar/operar.
- **Simplicidade agora vs. escalabilidade futura**: manter o worker como instância única simplifica a garantia de ordenação hoje, mas limita a escalabilidade horizontal sem trabalho adicional futuro de particionamento ou locking.

## Relação com o sistema existente

- `src/server.ts` é o padrão de referência para o futuro `src/worker.ts`: bootstrap do processo, tratamento de `SIGINT`/`SIGTERM` com shutdown gracioso e desconexão do Prisma (`prisma.$disconnect()`).
- `src/config/database.ts` define o padrão atual de criação do `PrismaClient` (`createPrismaClient()`); o worker deve reaproveitar esse padrão de criação para instanciar seu próprio client, e não o singleton compartilhado pela API.
- `package.json` atual **não** possui um script `worker` nos scripts declarados (`dev`, `build`, `start`, `db:migrate`, `db:reset`, `db:seed`, `test`, `test:watch`, `lint`, `format`) — o script `npm run worker` é proposto, ainda não existe.
- Uma busca no repositório confirma que nenhum arquivo `worker*` ou `webhook*` existe hoje; todos os elementos descritos nesta decisão (entry point, processador, módulo) são propostos.

## Rastreabilidade

| Origem | IDs |
| --- | --- |
| Decisões (transcrição) | DEC-003, DEC-004, DEC-005, DEC-006, DEC-018 |
| Alternativa descartada | ALT-003 |
| Requisitos relacionados | NFR-002, NFR-010, NFR-012, NFR-014 |
| Limitações conhecidas | LIMIT-001, LIMIT-002, LIMIT-005 |
| Risco mitigado | RISK-006 |
| Pontos de integração de código | CODE-010, CODE-011 |
| Timestamps-chave | `[09:08]-[09:14]`, `[09:28]-[09:30]` |

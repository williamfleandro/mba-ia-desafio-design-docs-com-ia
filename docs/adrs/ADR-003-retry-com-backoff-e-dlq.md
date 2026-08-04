# ADR-003: Retry com backoff exponencial e Dead Letter Queue (DLQ)

## Status

Aceita — decidida em reunião técnica; implementação ainda não iniciada (ver "Relação com o sistema existente").

## Data

Reunião técnica de quinta-feira, 09:00 (registrada em `TRANSCRICAO.md`; a transcrição não informa a data absoluta, apenas o dia da semana e o horário).

## Decisores

- Diego (Engenheiro Sênior, time de Plataforma) — propõe backoff, número de tentativas e DLQ
- Larissa (Tech Lead) — fecha as decisões
- Bruno (Engenheiro Pleno, time de Pedidos) — questiona número de tentativas e reprocessamento
- Sofia (Engenheira de Segurança) — levanta a exigência de timeout do HTTP call (valor de 10s definido por Diego) e exige `ADMIN` + auditoria no replay

## Fontes

- `TRANSCRICAO.md`, `[09:14]-[09:19]`, `[09:35]-[09:36]`, `[09:42]`
- `reports/context-analysis.md`, decisões DEC-007, DEC-008, DEC-009, DEC-010, DEC-023, DEC-028; alternativas ALT-004, ALT-005, ALT-006; risco RISK-004; NFR-007, NFR-008; requisitos FR-009, FR-010, FR-011
- Código: `src/middlewares/auth.middleware.ts` (`authenticate`, `requireRole`)

## Contexto

Diego levantou a necessidade de uma política de retentativas para quando o cliente do webhook estiver offline ou indisponível (`[09:14] Larissa`, `[09:15] Diego`). A discussão precisava equilibrar dois riscos: eventos "pendurados para sempre" se o cliente sumir definitivamente, e descarte prematuro de eventos recuperáveis se o número de tentativas fosse baixo demais — a equipe citou um caso real de cliente com 2 horas de indisponibilidade por manutenção planejada (`[09:16] Diego`).

## Decisão

- Retentativas com **backoff exponencial**, com um teto de **5 tentativas** (`[09:15]-[09:17] Diego, Larissa`).
- Progressão do backoff: **1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas**, totalizando aproximadamente 15 horas entre a primeira falha e a última tentativa (`[09:17] Diego`).
- O **timeout do HTTP call** feito pelo worker é de **10 segundos**; um cliente que não responde nesse prazo é tratado como falha e entra no fluxo de retry (`[09:42] Diego, Sofia`).
- Após a 5ª tentativa falhar, o evento é movido para uma **tabela separada** (proposta como `webhook_dead_letter`), guardando payload, motivo da falha e timestamp — mantendo a outbox principal limpa para leitura do worker, e servindo como evidência para debug e reprocessamento (`[09:17]-[09:18] Diego`).
- O reprocessamento de eventos em DLQ é **manual**, via um endpoint administrativo: `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento como pendente na outbox (`[09:18]-[09:19] Diego, Larissa`). Esse endpoint exige role `ADMIN` (via `requireRole` já existente) e deve logar quem executou o replay, para fins de auditoria (`[09:35]-[09:36] Sofia, Larissa`).

## Alternativas consideradas

- **Retry indefinido, sem teto de tentativas** (`[09:15] Diego`): descartado porque deixaria eventos pendurados para sempre caso o cliente do webhook tivesse desaparecido definitivamente.
- **3 tentativas** (`[09:16] Bruno, Diego`): descartado por ser insuficiente frente a indisponibilidades reais já observadas — um cliente com 2h de manutenção planejada teria as 3 tentativas esgotadas em ~30 minutos, descartando o evento cedo demais.
- **Marcar falha permanente como status "failed" na própria tabela de outbox** (`[09:17]-[09:18] Diego`): descartada em favor de uma tabela separada, que mantém a leitura da outbox principal mais limpa e funciona como evidência isolada para debug/reprocessamento.

## Consequências positivas

- Cobre janelas realistas de indisponibilidade do cliente (~15h), baseadas em um incidente real observado pela equipe.
- A DLQ separada é auditável e reprocessável sem poluir a tabela de leitura principal do worker.
- O timeout de 10s evita que um cliente lento prenda o processamento do worker indefinidamente.
- O replay exige role `ADMIN` e log de auditoria, reduzindo o risco de reprocessamento não rastreável (`RISK-008` em `reports/context-analysis.md`).

## Consequências negativas

- Um evento pode levar até ~15 horas para ser declarado falha permanente e migrar para a DLQ — não é uma resposta rápida a falhas definitivas.
- Mais uma tabela (`webhook_dead_letter`) para manter, monitorar e operar.
- Reprocessamento de eventos em DLQ é estritamente manual — não há retry automático a partir da DLQ; depende de intervenção humana via endpoint admin.

## Trade-offs

- **Janela de recuperação vs. tempo até declarar falha permanente**: 5 tentativas com backoff longo cobrem mais cenários de indisponibilidade temporária, à custa de eventos ficarem "em aberto" por mais tempo antes de ir para a DLQ.
- **Limpeza da outbox vs. superfície de schema**: separar a DLQ em tabela própria mantém a outbox principal enxuta para o worker, mas adiciona uma tabela extra ao domínio.
- **Automação vs. controle**: reprocessamento manual via endpoint admin garante uma ação explícita e auditável, ao custo de não haver recuperação automática de eventos em DLQ.

## Relação com o sistema existente

- O endpoint de replay deve reaproveitar diretamente `authenticate` e `requireRole('ADMIN')`, já definidos em `src/middlewares/auth.middleware.ts:27-61` — nenhuma alteração é necessária nesse middleware.
- Erros específicos deste fluxo (ex.: evento de DLQ não encontrado) devem seguir o padrão `AppError` com prefixo `WEBHOOK_`, tratado pelo `errorMiddleware` já existente (ver ADR-006).
- A tabela `webhook_dead_letter` e o endpoint `POST /admin/webhooks/dead-letter/:id/replay` são elementos propostos — não existem hoje no `prisma/schema.prisma` nem nas rotas do sistema.

## Rastreabilidade

| Origem | IDs |
| --- | --- |
| Decisões (transcrição) | DEC-007, DEC-008, DEC-009, DEC-010, DEC-023, DEC-028 |
| Alternativas descartadas | ALT-004, ALT-005, ALT-006 |
| Requisitos relacionados | FR-009, FR-010, FR-011, NFR-007, NFR-008 |
| Risco mitigado | RISK-004, RISK-008 |
| Pontos de integração de código | `src/middlewares/auth.middleware.ts` (`authenticate`, `requireRole`) |
| Timestamps-chave | `[09:14]-[09:19]`, `[09:35]-[09:36]`, `[09:42]` |

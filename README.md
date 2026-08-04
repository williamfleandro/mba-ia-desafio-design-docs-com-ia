# Da Reunião ao Documento: Design Docs Gerados por IA

## Sobre o desafio

Este repositório documenta a produção do pacote de design docs do **Sistema de Webhooks de Notificação de Pedidos**, uma feature discutida e fechada em uma reunião técnica de ~55 minutos entre tech lead, PM, dois engenheiros e uma engenheira de segurança, mas nunca registrada em nenhum documento — só existia a gravação em `TRANSCRICAO.md` e o código do OMS em produção. A tarefa foi transformar essa reunião em documentação acionável o suficiente para o time começar a implementar: PRD, RFC, FDD, seis ADRs e um Tracker de rastreabilidade, usando `TRANSCRICAO.md` e o código-fonte como as únicas fontes primárias de informação.

A entrega é puramente documental — em nenhum momento o código da aplicação (`src/`, `prisma/`, `tests/`, configurações) foi alterado; ele serviu exclusivamente como referência para mapear onde e como a feature se integraria ao sistema existente. O maior risco do processo não era técnico, era de **alucinação**: a IA podia facilmente inventar um endpoint, um nome de campo ou uma decisão que "soa plausível" mas nunca foi dita na reunião nem existe no código. Por isso, cada afirmação nos documentos precisava ser rastreável a um timestamp da transcrição (`[hh:mm] Nome`) ou a um caminho real de arquivo — e onde essa rastreabilidade não existia, o item foi marcado explicitamente como proposta sujeita a revisão, nunca apresentado como decisão fechada.

## Entregáveis

- [`docs/PRD.md`](./docs/PRD.md) — problema de negócio, público, escopo e critérios de sucesso da feature.
- [`docs/RFC.md`](./docs/RFC.md) — proposta técnica submetida à equipe para revisão: abordagem, alternativas descartadas e questões em aberto.
- [`docs/FDD.md`](./docs/FDD.md) — especificação de implementação: fluxos, contratos HTTP, matriz de erros, integração com o código.
- [`docs/TRACKER.md`](./docs/TRACKER.md) — tabela de rastreabilidade ligando cada item dos documentos à sua origem na transcrição ou no código.
- Seis ADRs em [`docs/adrs/`](./docs/adrs/) — uma decisão arquitetural isolada por arquivo, com contexto, alternativas e consequências.
- Este `README.md` — o relato do processo de produção.

## Ferramentas de IA utilizadas

### Claude Code

Ferramenta principal de produção. Rodou dentro do próprio repositório, com acesso direto ao sistema de arquivos, e foi usada para:

- ler a árvore de arquivos e o código-fonte do OMS (módulos, schema Prisma, middlewares, padrões de erro);
- analisar `TRANSCRICAO.md` linha a linha para extrair decisões, requisitos, alternativas e questões em aberto;
- gerar o conteúdo de cada documento (relatório de análise, ADRs, RFC, FDD, PRD, Tracker) e salvar os arquivos diretamente no repositório;
- revisar inconsistências factuais entre documentos (nomes de campos, autoria de decisões, classificação de itens);
- validar que todo caminho de arquivo citado nos documentos realmente existe no repositório, e que nenhum componente "futuro" fosse descrito como já existente.

### ChatGPT

Usado como camada de orquestração e revisão externa, fora do repositório, para:

- organizar o workflow geral e a ordem de produção dos documentos;
- elaborar os prompts dirigidos usados no Claude Code (em vez de pedidos genéricos do tipo "gere um PRD");
- revisar criticamente os resultados produzidos pelo Claude Code, comparando-os entre si;
- identificar possíveis contradições entre documentos (por exemplo, um mesmo dado descrito de forma diferente no PRD e no FDD);
- orientar validações pontuais com Git e PowerShell (conferência de arquivos gerados, encoding, diffs);
- garantir que a separação de altura entre PRD, RFC, FDD, ADRs e Tracker fosse respeitada — sinalizando quando um documento estava reproduzindo o nível de detalhe de outro.

## Workflow adotado

1. **Preparação do repositório** — fork do repositório base, clone local e criação da branch `feature/design-docs-webhooks`.
2. **Contextualização e extração de evidências** — inspeção da árvore de arquivos, leitura completa de `TRANSCRICAO.md` e do código-fonte relevante (módulo `orders`, schema Prisma, middlewares, padrão de erros), e produção de `reports/context-analysis.md`.
3. **Produção dos ADRs** — as seis decisões arquiteturais foram escritas antes do RFC e do FDD.
4. **Auditoria dos ADRs** — revisão dedicada, comparando cada afirmação de volta com a transcrição e o código.
5. **Produção do RFC** — consolidação da proposta técnica em cima das decisões já fechadas nos ADRs.
6. **Produção do FDD** — detalhamento de implementação a partir do RFC e dos ADRs.
7. **Produção do PRD** — escrito por último entre os documentos grandes, como consolidação de produto sobre o que já estava decidido tecnicamente.
8. **Correção pontual de consistência do PRD** — ajuste de uma frase ambígua sobre rollback (ver "Iterações e ajustes").
9. **Construção do Tracker** — varredura de todos os documentos prontos para montar a matriz de rastreabilidade.
10. **README** — escrito por último, com o processo já completo.
11. **Revisão final** — checklist de critérios de aceite item por item.

### Por que os ADRs vieram antes do RFC e do FDD

A reunião não discutiu a feature em blocos separados de "arquitetura" e "produto" — as decisões técnicas (outbox, worker, retry, HMAC, at-least-once, reuso de padrões) foram o conteúdo central de quase toda a call. Produzir os ADRs primeiro significou fixar essas decisões, uma por uma, com contexto e consequências, antes de tentar redigir qualquer prosa em cima delas. O RFC pôde então ser escrito como uma consolidação e um link para esses ADRs, em vez de redescobrir as mesmas decisões com outras palavras — e o FDD, por sua vez, herdou dos ADRs a base decisória sobre a qual detalhar contratos e fluxos. Sem essa ordem, o risco real era duplicar (e eventualmente divergir) a mesma decisão em três documentos diferentes.

## Prompts customizados

### Prompt 1 — Análise estruturada e rastreável

Prompt usado para produzir `reports/context-analysis.md`, o artefato intermediário que sustentou todos os documentos seguintes.

```text
Leia integralmente TRANSCRICAO.md e o código-fonte do repositório
(estrutura de módulos, schema Prisma, middlewares, padrão de erros).
Não altere nenhum arquivo em src/, prisma/, tests/ ou configurações.

Produza uma análise estruturada, separando claramente:
- decisões arquiteturais fechadas na reunião;
- requisitos funcionais e não funcionais;
- alternativas consideradas e descartadas, com o motivo do descarte;
- itens adiados ou explicitamente fora de escopo;
- questões deixadas em aberto (sem decisão fechada);
- limitações conhecidas aceitas pela equipe;
- pontos reais de integração com o código existente (caminho de
  arquivo + símbolo relevante).

Cada item deve citar sua origem: para transcrição, o timestamp exato
no formato [hh:mm] Nome; para código, o caminho real do arquivo.
Valide que todo caminho de arquivo citado realmente existe no
repositório antes de incluí-lo. Ao final, gere uma matriz de
evidências consolidada e confirme que nenhum item foi promovido de
"mencionado" para "decidido" sem base direta na transcrição.
```

### Prompt 2 — Produção e revisão dos ADRs

Prompt usado para gerar os seis ADRs a partir de `reports/context-analysis.md`, com uma auditoria explícita embutida.

```text
Com base em reports/context-analysis.md e TRANSCRICAO.md, produza um
ADR por decisão arquitetural isolada, em docs/adrs/, formato
ADR-NNN-titulo-em-kebab-case.md, seguindo MADR: Status, Contexto,
Decisão, Alternativas Consideradas, Consequências (positivas e
negativas, com trade-off explícito).

Regras:
- Toda alternativa citada deve ser uma alternativa real discutida
  na reunião, nunca inventada.
- Pelo menos um ADR deve referenciar arquivos, módulos ou padrões
  reais do código (caminho + linha, quando aplicável).
- Se uma característica (ex.: ordenação de eventos) foi tratada na
  reunião como consequência/limitação de uma decisão maior, e não
  como uma decisão independente com alternativas avaliadas, não crie
  um ADR separado para ela — registre como seção de consequência
  dentro do ADR da decisão da qual ela deriva.

Depois de gerar os seis ADRs, faça uma auditoria: releia cada
afirmação contra a transcrição e o código, e corrija qualquer nome
de campo, autoria de decisão ou referência de arquivo que não
confira exatamente com a fonte.
```

### Prompt 3 — Produção do FDD

Prompt usado para o documento de implementação, o mais técnico do pacote.

```text
Com base no RFC, nos seis ADRs e em reports/context-analysis.md,
produza docs/FDD.md como o documento acionável para um desenvolvedor
começar a codar. Inclua contratos HTTP completos (request/response
de exemplo, status codes) para pelo menos 4 endpoints, matriz de
erros com códigos no padrão WEBHOOK_*, fluxos detalhados (outbox,
worker, retry, DLQ), estratégias de resiliência e observabilidade
(métricas, logs, tracing).

Seção obrigatória "Integração com o sistema existente": nomeie pelo
menos 4 caminhos de arquivo reais do repositório e descreva como o
módulo de webhooks se integra a cada um.

Qualquer detalhe necessário para implementação que não tenha sido
fechado na reunião (nome de campo, path de endpoint, tamanho de
lote) deve ser proposto, mas sinalizado explicitamente como
"Decisão de design proposta no FDD, sujeita à revisão" — nunca
apresentado como se tivesse sido decidido na call.
```

## Iterações e ajustes

O processo levou aproximadamente **8 ciclos principais** de geração, revisão e refinamento: análise inicial → ADRs → auditoria dos ADRs → RFC → FDD → PRD (com uma correção pontual) → Tracker → README e revisão final. Dentro desses ciclos, quatro ajustes concretos valem registro:

### Ajuste 1 — Codificação no PowerShell

Ao inspecionar um dos documentos gerados usando `Get-Content` no PowerShell sem o parâmetro `-Encoding UTF8`, os acentos apareceram corrompidos no terminal. A suspeita inicial foi de que o arquivo tinha sido salvo com encoding errado. Uma inspeção mais cuidadosa — reabrindo o mesmo arquivo com `-Encoding UTF8` e também no editor — confirmou que o conteúdo em UTF-8 estava correto; o problema era exclusivamente a renderização do terminal, cujo codepage padrão não exibe corretamente caracteres UTF-8. O PowerShell foi então configurado para UTF-8 na sessão de trabalho, e a inspeção de arquivos passou a usar `-Encoding UTF8` de forma consistente.

### Ajuste 2 — ADR de ordenação não criado separadamente

Na primeira passada sobre `reports/context-analysis.md`, a ordenação de eventos condicionada a single-worker (`[09:12]-[09:13]`) foi cotada como candidata a um ADR próprio (`ADR-008`), por ter alternativas citadas (particionamento por `order_id`, lock pessimista) e consequências claras. Uma releitura da transcrição mostrou, porém, que Larissa tratou esse ponto explicitamente como **limitação conhecida** derivada do design já decidido no ADR do worker ("Documentamos como limitação conhecida. Não é garantia de ordering global, só por `order_id` e enquanto for single-worker", `[09:13]`), não como uma decisão independente com alternativas avaliadas e comparadas na reunião. A decisão final foi não criar um `ADR-007` de ordenação: o comportamento foi mantido como uma seção de consequência dentro do `ADR-002` (worker separado com polling), que é de onde ele efetivamente deriva.

### Ajuste 3 — Correções factuais nos ADRs

A auditoria dedicada aos seis ADRs (ciclo 3) encontrou e corrigiu três imprecisões introduzidas na primeira geração:

- o nome de campo do estoque estava grafado como `stock_quantity` (snake_case, como mencionado coloquialmente na fala de Bruno) e foi corrigido para `stockQuantity`, o nome real do campo em `prisma/schema.prisma` e em `order.service.ts`;
- a autoria da decisão de timeout de 10 segundos no HTTP call do worker estava atribuída de forma imprecisa e foi corrigida para Diego, que propõe o valor em `[09:42]`, com a concordância de Sofia;
- a confirmação do reuso do Prisma com instância própria por processo estava atribuída apenas a Diego e foi corrigida para refletir que Bruno confirma esse ponto explicitamente em `[09:29]-[09:30]`.

### Ajuste 4 — Consistência do PRD sobre rollback

Na seção de objetivos do PRD, uma formulação inicial poderia ser lida como "nenhuma falha relacionada ao webhook deve causar rollback da mudança de status", o que contradiz diretamente a decisão registrada em `ADR-001` e na transcrição (`[09:40]-[09:41] Bruno, Diego`). Foi necessário distinguir dois tipos de falha que a reunião trata de forma bem diferente: a **chamada HTTP externa** ao cliente (que nunca deve bloquear nem reverter a transação de status, por rodar de forma assíncrona no worker) e a **inserção do evento na outbox** (que faz parte da mesma transação atômica de `changeStatus` e, se falhar, deve sim provocar rollback — essa é justamente a garantia central do padrão Outbox). O texto do PRD foi ajustado para deixar essa distinção explícita.

## Decisões de processo

- **Fontes primárias**: apenas `TRANSCRICAO.md` e o código-fonte do repositório. Nenhuma outra fonte (documentação externa, conhecimento geral sobre webhooks, práticas "de mercado" não citadas na reunião) foi tratada como base de decisão.
- **`reports/context-analysis.md` é artefato intermediário, não fonte primária**: o Tracker aponta sempre para `TRANSCRICAO` ou `CODIGO`, nunca para o relatório de análise — ele existe para organizar a extração de evidências, não para ser citado como origem de nada.
- **Propostas marcadas como propostas**: todo elemento necessário para a implementação, mas não fechado na reunião (nome de campo, contrato exato de um endpoint, tamanho de lote do worker), foi sinalizado no FDD como "Decisão de design proposta no FDD, sujeita à revisão", e listado separadamente no Tracker como item sem decisão fechada — nunca apresentado como se tivesse sido decidido na call.
- **Nenhuma informação sem fonte foi promovida a requisito ou decisão**: itens apenas mencionados de passagem na reunião (ex.: rate limiting) permaneceram como questão em aberto, não viraram requisito funcional.
- **Distribuição por altura documental**: cada detalhe foi colocado no documento cuja altura lhe cabe — problema de negócio no PRD, proposta e alternativas no RFC, decisão isolada no ADR correspondente, detalhe de implementação no FDD — evitando duplicar o mesmo conteúdo em profundidades diferentes.
- **Ordenação não virou ADR separado**: por ser tratada na transcrição como limitação derivada, e não como decisão independente (ver "Ajuste 2").
- **O Tracker como mecanismo anti-alucinação**: a exigência de preencher a coluna "Localização" para cada linha funcionou como filtro prático — qualquer item para o qual não havia timestamp ou caminho de código correspondente era, por definição, uma invenção e precisava ser removido ou reclassificado como proposta.

## Como navegar pela entrega

1. **`docs/PRD.md`** — comece aqui para entender o problema de negócio, os três clientes que motivaram a feature, o escopo e os critérios de aceitação.
2. **`docs/RFC.md`** — leia em seguida para ver a proposta técnica consolidada, as alternativas que foram descartadas e por quê, e as questões que seguem em aberto.
3. **`docs/adrs/`** — percorra os seis ADRs para entender cada decisão arquitetural isoladamente, com seu contexto, trade-offs e relação com o código existente.
4. **`docs/FDD.md`** — vá aqui para o nível de implementação: contratos HTTP, matriz de erros `WEBHOOK_*`, fluxos detalhados e a seção "Integração com o sistema existente".
5. **`docs/TRACKER.md`** — use como referência cruzada final, para verificar a origem exata (transcrição ou código) de qualquer afirmação encontrada nos documentos anteriores.

## Estrutura do repositório

```text
.
├── README.md
├── TRANSCRICAO.md
├── docs/
│   ├── PRD.md
│   ├── RFC.md
│   ├── FDD.md
│   ├── TRACKER.md
│   └── adrs/
│       ├── ADR-001-outbox-transacional-no-mysql.md
│       ├── ADR-002-worker-separado-com-polling.md
│       ├── ADR-003-retry-com-backoff-e-dlq.md
│       ├── ADR-004-autenticacao-hmac-sha256.md
│       ├── ADR-005-entrega-at-least-once-com-event-id.md
│       └── ADR-006-reuso-dos-padroes-existentes.md
├── reports/
│   └── context-analysis.md
├── src/
├── prisma/
└── tests/
```

`src/`, `prisma/` e `tests/` contêm a aplicação OMS existente e foram usados exclusivamente como **contexto e referência** durante a produção dos documentos — nenhum arquivo dentro dessas pastas foi criado, modificado ou removido neste processo.

## Critérios de qualidade aplicados

- Rastreabilidade de cada requisito, decisão e restrição a um timestamp de `TRANSCRICAO.md` ou a um caminho real de código.
- Ausência de contradições entre PRD, RFC, FDD e ADRs (verificada explicitamente na auditoria e no ajuste do PRD sobre rollback).
- Separação de responsabilidades documentais: cada documento opera na sua própria altura, sem duplicar o nível de detalhe de outro.
- Validação de que todo caminho de arquivo citado existe de fato no repositório.
- Distinção clara, em todo o pacote, entre o que já existe no código e o que é proposto para implementação futura.
- Revisão de nomes de participantes e timestamps contra a transcrição completa, linha a linha.
- Cobertura do Tracker acima do mínimo exigido, com linhas de origem tanto em transcrição quanto em código.
- Consistência dos códigos de erro `WEBHOOK_*` entre o FDD e o Tracker.
- Nenhuma alteração em `src/`, `prisma/`, `tests/` ou arquivos de configuração da aplicação.

## Limitações e pontos ainda abertos

O pacote de documentos registra, de forma consciente, um conjunto de pontos que a reunião não fechou — nenhum deles foi resolvido neste README, apenas documentados em seus respectivos lugares (RFC e FDD):

- Rate limiting de envio outbound ao cliente, discutido mas não implementado nesta fase.
- Estratégia de arquivamento da outbox após eventos entregues.
- Escalonamento para múltiplos workers e o que isso exige para preservar ordenação.
- Armazenamento seguro (em repouso) das secrets de HMAC.
- Escolha da biblioteca/abstração HTTP para as chamadas do worker.
- Definição formal de SLO operacional além do limiar de latência já decidido.
- Contrato final (método e path) do endpoint de rotação de secret.

## Autor

William Ferreira Leandro
Projeto acadêmico — MBA em Engenharia de Software com IA

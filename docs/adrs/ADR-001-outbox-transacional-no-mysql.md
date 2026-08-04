# ADR-001: Padrão Outbox transacional no MySQL para publicação de eventos de webhook

## Status

Aceita — decidida em reunião técnica; implementação ainda não iniciada (ver "Relação com o sistema existente").

## Data

Reunião técnica de quinta-feira, 09:00 (registrada em `TRANSCRICAO.md`; a transcrição não informa a data absoluta, apenas o dia da semana e o horário).

## Decisores

- Diego (Engenheiro Sênior, time de Plataforma) — proponente do padrão Outbox
- Larissa (Tech Lead) — fecha a decisão
- Bruno (Engenheiro Pleno, time de Pedidos) — valida impacto na transação de `changeStatus`

## Fontes

- `TRANSCRICAO.md`, `[09:03]-[09:08]`, `[09:33]-[09:34]`, `[09:40]-[09:42]`, `[09:51]-[09:52]`
- `reports/context-analysis.md`, decisões DEC-002, DEC-021, DEC-022, DEC-026, DEC-030, DEC-031; alternativas ALT-001, ALT-002; riscos RISK-001, RISK-002, RISK-005; pontos de integração CODE-001, CODE-002
- Código: `src/modules/orders/order.service.ts:126-179` (método `changeStatus`), `prisma/schema.prisma`

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pediram para ser notificados quando o status de seus pedidos muda, hoje resolvido via polling manual em `GET /orders` (`[09:00] Marcos`). A equipe discutiu como disparar essas notificações a partir da mudança de status em `OrderService.changeStatus`, que já executa, dentro de uma única transação Prisma (`this.prisma.$transaction`), a atualização do pedido, a inserção em `order_status_history` e o ajuste de `stockQuantity` (`src/modules/orders/order.service.ts:126-179`).

Bruno apontou que essa transação já é pesada e que um `HTTP call` síncrono no meio dela travaria mudanças de status de outros pedidos caso o cliente do webhook estivesse lento ou fora do ar, além de não haver uma forma limpa de decidir sobre rollback nesse cenário (`[09:04] Bruno`). Diego propôs o padrão Outbox: inserir uma linha de evento em uma tabela (referida como `webhook_outbox`) dentro da mesma transação SQL que já altera `orders` e `order_status_history`, deixando um worker separado ler essa tabela e disparar as chamadas HTTP (`[09:06] Diego`).

## Decisão

Adotar o padrão Outbox transacional em uma tabela MySQL (proposta como `webhook_outbox`, nome ainda não confirmado formalmente — ver `reports/context-analysis.md`, seção U.1):

- A inserção do evento de webhook ocorre **dentro da mesma transação Prisma** de `OrderService.changeStatus`, após a criação do registro em `order_status_history`. Se a inserção do evento falhar, toda a transação sofre rollback — não pode existir caso em que o status mude sem o evento ser registrado, ou vice-versa (`[09:40]-[09:41] Bruno, Diego`).
- A integração é feita por uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o `tx` (client de transação) já aberto por `changeStatus`, evitando injetar um repository inteiro no `OrderService` (`[09:41] Bruno, Diego`).
- O evento só é inserido na outbox se **algum** webhook cadastrado pelo customer estiver inscrito naquele status de destino; o filtro por status é aplicado na inserção, não no momento do envio, para não acumular linhas que nunca serão entregues (`[09:33]-[09:34] Bruno, Diego`).
- O payload do evento é gravado como um **snapshot renderizado no momento da inserção** (não apenas `order_id` a ser resolvido depois), para que o evento sempre reflita o estado do pedido no instante da mudança de status, mesmo que o pedido mude posteriormente (`[09:51]-[09:52] Larissa, Diego, Bruno`).
- Identificadores da outbox usam UUID, seguindo o padrão já usado em todo o schema Prisma atual (`[09:51] Larissa`).
- A tabela tem índice em `status` e em `created_at`; o worker lê apenas os eventos pendentes mais antigos, em lote pequeno (`[09:07]-[09:08] Diego`). Arquivamento de linhas já entregues (após ~30 dias) fica explicitamente fora do escopo desta feature (`[09:08] Diego`).

## Alternativas consideradas

- **Disparo síncrono dentro da transação de `changeStatus`** (`[09:03]-[09:05] Bruno, Larissa`): descartado porque um cliente de webhook lento ou indisponível travaria a transação de mudança de status de outros pedidos, e não há forma limpa de fazer rollback caso o cliente esteja fora do ar.
- **Fila externa dedicada, ex. Redis Streams** (`[09:07] Larissa, Diego`): descartada por ser overengineering para um time pequeno — exigiria subir e operar infraestrutura nova (ex. Redis Cluster) quando o MySQL já existente resolve o problema.

## Consequências positivas

- Atomicidade garantida entre a mudança de status do pedido e o registro do evento de webhook — elimina o risco de inconsistência descrito em `RISK-002`.
- Nenhuma infraestrutura nova é necessária; reaproveita o MySQL e o Prisma já usados pelo resto do sistema.
- O snapshot do payload na inserção evita que eventos entreguem dados desatualizados ou inconsistentes com o estado do pedido no momento da mudança.
- O filtro de status na inserção evita acumular linhas de evento que nenhum cliente quer receber.

## Consequências negativas

- A lógica de publicação de webhook fica acoplada à transação crítica de mudança de status de pedidos (`changeStatus`); qualquer falha na inserção do evento reverte toda a transação de negócio, inclusive ajustes de estoque.
- A tabela de outbox cresce continuamente; o arquivamento de eventos já entregues foi deliberadamente deixado fora do escopo desta feature (`OUT-003`), ficando como dívida futura.
- Introduz dependência de leitura eficiente (índices em `status`/`created_at`) para que o acúmulo de eventos não degrade a performance do worker (`RISK-005`).

## Trade-offs

- **Consistência forte vs. desacoplamento**: optou-se deliberadamente por acoplar a inserção do evento à transação de negócio ("Se ficar fora da transação, perde a garantia toda", `[09:41] Diego`), abrindo mão de um desacoplamento mais simples em favor de garantir que não exista inconsistência entre status e evento.
- **MySQL existente vs. infraestrutura de mensageria dedicada**: opta-se por reuso de infraestrutura já operada pelo time em vez dos recursos prontos (persistência distribuída, particionamento nativo) que uma fila dedicada ofereceria.

## Relação com o sistema existente

- Ponto de integração real: `src/modules/orders/order.service.ts:126-179`, método `changeStatus` — a chamada a `publishWebhookEvent(tx, order, from, to)` deveria ocorrer logo após `tx.orderStatusHistory.create(...)` (linha ~167) e antes do retorno do pedido atualizado.
- `prisma/schema.prisma` hoje define `Order`, `OrderStatusHistory` e o enum `OrderStatus`, que fornecem os campos usados no payload do evento (`order_id`, `order_number`, `from_status`/`to_status`, `customer_id`, `totalCents`). Novos modelos Prisma para a outbox (e para a configuração de webhook) ainda **não existem** no schema — são propostos.
- A função `publishWebhookEvent` e o módulo `src/modules/webhooks/` são elementos futuros propostos; uma busca no repositório (`webhook*`, `worker*`) não retornou nenhum arquivo existente até o momento desta análise.

## Rastreabilidade

| Origem | IDs |
| --- | --- |
| Decisões (transcrição) | DEC-002, DEC-021, DEC-022, DEC-026, DEC-030, DEC-031 |
| Alternativas descartadas | ALT-001, ALT-002 |
| Riscos mitigados | RISK-001, RISK-002, RISK-005 |
| Requisitos relacionados | FR-008, NFR-013, NFR-015 |
| Pontos de integração de código | CODE-001, CODE-002 |
| Timestamps-chave | `[09:03]-[09:08]`, `[09:33]-[09:34]`, `[09:40]-[09:42]`, `[09:51]-[09:52]` |

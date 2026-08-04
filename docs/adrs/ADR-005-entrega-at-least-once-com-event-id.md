# ADR-005: Garantia de entrega at-least-once com deduplicação via X-Event-Id

## Status

Aceita — decidida em reunião técnica; implementação ainda não iniciada (ver "Relação com o sistema existente").

## Data

Reunião técnica de quinta-feira, 09:00 (registrada em `TRANSCRICAO.md`; a transcrição não informa a data absoluta, apenas o dia da semana e o horário).

## Decisores

- Diego (Engenheiro Sênior, time de Plataforma) — propõe at-least-once e `X-Event-Id`
- Sofia (Engenheira de Segurança) — sugere `X-Webhook-Id` adicional
- Marcos (Product Manager) — assume documentar a exigência de dedup no portal do desenvolvedor

## Fontes

- `TRANSCRICAO.md`, `[09:24]-[09:26]`, `[09:43]-[09:45]`
- `reports/context-analysis.md`, decisões DEC-016, DEC-024, DEC-025; alternativa ALT-008; requisito FR-013; NFR-009; limitação LIMIT-003
- Código: nenhum artefato existente — funcionalidade inteiramente nova (confirmado por busca de `webhook*`/`worker*` no repositório)

## Contexto

Definida a arquitetura de outbox e worker (ADR-001, ADR-002) e o mecanismo de retry (ADR-003), a equipe discutiu qual garantia de entrega oferecer aos clientes, já que retentativas podem resultar em uma mesma chamada HTTP sendo recebida mais de uma vez pelo cliente (`[09:24] Diego`).

## Decisão

- O sistema garante entrega **at-least-once** (pelo menos uma vez), não exactly-once: o cliente pode receber o mesmo evento mais de uma vez e deve estar preparado para isso (`[09:24]-[09:25] Diego`).
- Cada evento recebe um **UUID `event_id`**, gerado no momento em que o evento entra na outbox (ADR-001), único por evento, enviado no header **`X-Event-Id`**. O cliente é responsável por deduplicar localmente com base nesse identificador (`[09:25] Diego`).
- Demais headers definidos para o envio: `X-Timestamp` (timestamp do envio, permitindo ao cliente detectar replay attacks), `X-Webhook-Id` (identifica qual cadastro de endpoint webhook do cliente disparou aquele envio — útil para clientes com múltiplos endpoints, sugerido por Sofia), `Content-Type: application/json`, e `X-Signature` (definido em ADR-004) (`[09:44]-[09:45] Diego, Sofia`).
- O formato do payload é JSON com `event_id`, `event_type` (`"order.status_changed"`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`; o campo `items` é explicitamente excluído para manter o payload enxuto — o cliente busca detalhes completos via `GET /orders/:id` se precisar (`[09:43] Diego, Bruno`).
- Marcos se comprometeu a documentar essa exigência de deduplicação de forma destacada no portal do desenvolvedor para os clientes (`[09:26] Marcos`).

## Alternativas consideradas

- **Garantia de entrega exactly-once** (`[09:24]-[09:26] Diego`): descartada porque exigiria coordenação complexa entre as duas partes (sistema e cliente) para eliminar duplicidade de forma definitiva; Diego argumentou que at-least-once com `event_id` já é o padrão de mercado adotado por provedores como Stripe e GitHub, e resolve "99% dos casos" com muito menos complexidade de implementação e operação.

## Consequências positivas

- Simplicidade de implementação e operação do lado do sistema — não é necessário nenhum mecanismo de coordenação distribuída para eliminar duplicatas.
- Consistente com a arquitetura de retry/DLQ (ADR-003), que por natureza pode gerar tentativas de reenvio de um evento já processado parcialmente pelo cliente.
- Alinhado a um padrão de mercado já conhecido por integradores de API (citados Stripe e GitHub), reduzindo a curva de aprendizado do cliente.

## Consequências negativas

- Transfere a responsabilidade de deduplicação para o cliente — Sofia observou explicitamente que "isso joga responsabilidade pro cliente" (`[09:25] Sofia`).
- Depende de documentação clara e visível no portal do desenvolvedor para que os clientes efetivamente implementem a deduplicação; falha do cliente em implementar isso pode gerar efeitos colaterais duplicados do lado dele (não mitigável pelo sistema).

## Trade-offs

- **Simplicidade operacional vs. garantia mais forte**: at-least-once é mais simples e barato de operar do que exactly-once, mas exige que o cliente implemente lógica de deduplicação — uma responsabilidade que, com exactly-once, ficaria concentrada no sistema.
- **Payload enxuto vs. completude**: excluir `items` do payload reduz custo de banda e de renderização, mas exige uma chamada adicional (`GET /orders/:id`) do cliente caso precise dos detalhes completos do pedido.

## Relação com o sistema existente

- O `event_id` é gerado no momento da inserção do evento na outbox, dentro da transação de `OrderService.changeStatus` (ver ADR-001) — não há hoje geração de UUID de evento em nenhum lugar do código, pois a funcionalidade é inteiramente nova.
- Os headers de envio (`X-Event-Id`, `X-Timestamp`, `X-Webhook-Id`, `X-Signature`, `Content-Type`) são adicionados pelo worker no momento da chamada HTTP (ver ADR-002) — nenhum código de envio de webhook existe hoje no repositório.
- Nenhuma estratégia de versionamento do schema do evento foi discutida na reunião; não deve ser assumida ou inventada.

## Rastreabilidade

| Origem | IDs |
| --- | --- |
| Decisões (transcrição) | DEC-016, DEC-024, DEC-025 |
| Alternativa descartada | ALT-008 |
| Requisitos relacionados | FR-013, NFR-009 |
| Limitação conhecida | LIMIT-003 |
| Relação com outros ADRs | ADR-001 (origem do `event_id`), ADR-002 (envio pelo worker), ADR-004 (`X-Signature`) |
| Timestamps-chave | `[09:24]-[09:26]`, `[09:43]-[09:45]` |

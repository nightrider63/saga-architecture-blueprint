# Decisões Arquiteturais — Saga de Checkout

## 1. Decisão e contexto

**Decisão:** Saga por orquestração, coordenada por um componente lógico chamado **Saga Orchestrator**. Order, Inventory e Payment mantêm bancos próprios e realizam apenas transações locais. A comunicação entre serviços acontece por comandos e eventos em um broker durável.

O objetivo é manter consistência eventual, evitar uma transação distribuída ACID e permitir que falhas posteriores acionem compensações explícitas.

### Por que orquestração, e não coreografia?

| Critério | Orquestração (escolhida) | Coreografia (alternativa) |
|---|---|---|
| Visibilidade do fluxo | Estado e transições observáveis em um ponto lógico | O fluxo emerge de eventos espalhados pelos serviços |
| Compensação | O coordenador conhece o passo atual e os passos a compensar | Cada consumidor precisa conhecer eventos e responsabilidades de compensação |
| Acoplamento | Serviços são acoplados a contratos de comando/evento; a ordem fica no orquestrador | Menos coordenador central, porém dependências semânticas entre produtores e consumidores podem se espalhar |
| Complexidade operacional | Um componente adicional e sua persistência precisam ser operados | Menos componente central, mas rastrear fluxo longo pode ficar difícil |
| Evolução | Mais simples visualizar políticas de timeout, retry e estados globais | Pode funcionar bem em fluxos curtos e altamente independentes |
| Escalabilidade | Orquestrador pode ser replicado com controle de concorrência e estado persistente | Consumidores e broker escalam independentemente |

Como o exercício exige compensações explícitas entre três domínios e um fluxo crítico de compra, a orquestração torna as transições, os prazos e o processo de recuperação mais fáceis de auditar. O broker e os consumidores continuam horizontalmente escaláveis; a escolha não exige processamento síncrono centralizado.

## 2. Componentes e responsabilidades

- **API Gateway / Checkout API:** autenticação, validação de entrada, limites por cliente e encaminhamento da solicitação.
- **Order Service:** cria o pedido como `PENDING`, gerencia `CONFIRMED` ou `CANCELLED` e mantém seu próprio banco.
- **Saga Orchestrator:** executa a máquina de estados, persiste a etapa, programa timeouts, gera comandos e coordena compensações.
- **Inventory Service:** reserva unidades por um TTL, confirma ou libera uma reserva e evita overselling por meio de atualização atômica da disponibilidade.
- **Payment Service:** processa o pagamento com o gateway externo e mantém a referência, o estado e a chave de idempotência da transação.
- **Message Broker:** entrega comandos e eventos de modo assíncrono, com confirmação, redelivery e filas de dead-letter configuradas.
- **Outbox/Inbox:** garantem publicação confiável e deduplicação na fronteira de cada serviço.
- **Observabilidade:** métricas, logs estruturados, traces distribuídos, alertas e trilha de auditoria por `sagaId` e `correlationId`.

## 3. Contratos de comunicação

As mensagens devem ser versionadas e incluir, no mínimo:

```json
{
  "messageId": "UUID único da mensagem",
  "messageType": "ReserveStock",
  "schemaVersion": 1,
  "sagaId": "UUID da saga",
  "correlationId": "UUID da jornada de compra",
  "causationId": "UUID da mensagem que causou esta mensagem",
  "orderId": "identificador do pedido",
  "occurredAt": "timestamp ISO-8601",
  "payload": {}
}
```

`messageId`, `sagaId`, `correlationId`, `causationId`, timestamps e esquema devem ser consistentes em todos os comandos/eventos. Payloads devem transportar somente os dados necessários, sem dados de cartão ou segredos do gateway.

### Comandos e eventos principais

| Mensagem | Tipo | Produtor → consumidor | Semântica |
|---|---|---|---|
| `OrderCreated` | Evento | Order Service → Broker → Orchestrator | Pedido aceito e persistido como `PENDING` |
| `ReserveStock` | Comando | Orchestrator → Inventory Service | Criar reserva temporária idempotente |
| `StockReserved` | Evento | Inventory Service → Orchestrator | Reserva criada, com `reservationId` e expiração |
| `StockReservationFailed` | Evento | Inventory Service → Orchestrator | Reserva negada por indisponibilidade ou regra de negócio |
| `ProcessPayment` | Comando | Orchestrator → Payment Service | Processar pagamento com chave idempotente estável |
| `PaymentSucceeded` | Evento | Payment Service → Orchestrator | Resultado financeiro confirmado pelo gateway |
| `PaymentDeclined` | Evento | Payment Service → Orchestrator | Recusa definitiva do gateway |
| `PaymentStatusUnknown` | Evento/estado | Payment Service → Orchestrator | Resultado incerto; requer reconciliação |
| `ConfirmOrder` | Comando | Orchestrator → Order Service | Transicionar pedido para `CONFIRMED` |
| `CancelOrder` | Comando | Orchestrator → Order Service | Transicionar pedido para `CANCELLED` |
| `ReleaseStock` | Comando | Orchestrator → Inventory Service | Liberar reserva ainda não consumida |
| `StockReleased` | Evento | Inventory Service → Orchestrator | Compensação de estoque concluída |
| `OrderConfirmed` / `OrderCancelled` | Evento | Order Service → Orchestrator | Resultado persistido do pedido |

A resposta HTTP inicial pode ser `202 Accepted`, com `orderId` e URL para consultar o status. Não se deve manter a conexão HTTP aberta aguardando todas as etapas assíncronas.

## 4. Máquina de estados da Saga

Estados sugeridos:

- `STARTED`
- `RESERVING_STOCK`
- `PROCESSING_PAYMENT`
- `PAYMENT_UNKNOWN`
- `CONFIRMING_ORDER`
- `COMPENSATING`
- `COMPLETED`
- `COMPENSATED`
- `MANUAL_REVIEW` (somente para inconsistência que não possa ser reconciliada automaticamente)

Transições típicas:

```text
STARTED -> RESERVING_STOCK
RESERVING_STOCK -- StockReserved --> PROCESSING_PAYMENT
RESERVING_STOCK -- StockReservationFailed --> COMPENSATING/CANCEL_ORDER
PROCESSING_PAYMENT -- PaymentSucceeded --> CONFIRMING_ORDER
PROCESSING_PAYMENT -- PaymentDeclined --> COMPENSATING
PROCESSING_PAYMENT -- timeout sem resultado definitivo --> PAYMENT_UNKNOWN
PAYMENT_UNKNOWN -- reconciliação confirma sucesso --> CONFIRMING_ORDER
PAYMENT_UNKNOWN -- reconciliação confirma recusa --> COMPENSATING
CONFIRMING_ORDER -- OrderConfirmed --> COMPLETED
COMPENSATING -- todas as compensações confirmadas --> COMPENSATED
qualquer estado -- inconsistência não reconciliável --> MANUAL_REVIEW
```

Na recusa definitiva do pagamento, `CancelOrder` e `ReleaseStock` podem ser emitidos em paralelo. A Saga só termina como `COMPENSATED` quando as duas compensações tiverem confirmação persistida. Se a etapa de pagamento tiver sido bem-sucedida e uma etapa posterior tornar a compra inviável, é necessária compensação financeira explícita (`VoidPayment` ou `RefundPayment`, conforme o estágio financeiro) antes de marcar a recuperação como concluída.

## 5. Matriz de idempotência e reentrega

| Operação | Chave idempotente / restrição | Comportamento em duplicidade |
|---|---|---|
| Criar pedido | `Idempotency-Key` do cliente + restrição única por cliente/operação | Retornar o mesmo `orderId` e o resultado original |
| Consumir mensagem | `messageId` em Inbox com índice único | Registrar processamento uma vez; ACK de cópias repetidas sem reaplicar efeito |
| Reservar estoque | `sagaId + orderId + itemId` ou `reservationId` estável | Retornar a reserva existente, sem descontar unidades novamente |
| Processar pagamento | Chave idempotente persistida e enviada ao gateway | Reutilizar a mesma chave em retries; consultar resultado antes de submeter nova transação |
| Confirmar pedido | `orderId` + transição de estado válida | `CONFIRMED` repetido não gera efeito adicional |
| Cancelar pedido | `orderId` + estado/versionamento | Pedido já cancelado permanece cancelado; não reabre transição terminal |
| Liberar estoque | `reservationId` | Reserva já liberada não devolve unidades novamente |
| Compensar pagamento | `paymentId + compensationType` | Evitar duplicação de void/refund; consultar gateway e guardar a referência de compensação |

Os consumidores devem persistir o registro da Inbox e o efeito de negócio na mesma transação local quando ambos residirem no mesmo banco. A entrega do broker pode ser *at-least-once*; a correção depende da deduplicação e da idempotência, não de pressupor entrega exatamente uma vez.

## 6. Estado persistente, timeouts e falhas de rede

A tabela de Saga deve guardar `sagaId`, `orderId`, `state`, `currentStep`, versões, timestamps, contador de tentativas, `deadlineAt`, IDs de mensagens/comandos, resultado da última etapa, detalhes do erro e status de cada compensação. Atualizações concorrentes devem usar versionamento otimista ou bloqueio apropriado.

**Timeout de transporte não comprova falha de negócio.** Em especial, se o gateway demorar após aceitar um pagamento, a resposta pode se perder. O orquestrador deve mover a Saga para `PAYMENT_UNKNOWN`, consultar o Payment Service/gateway e repetir a operação apenas com a mesma chave de idempotência. O estoque não deve ser liberado enquanto existir risco de pagamento aprovado sem compensação financeira.

Estratégia recomendada:

1. Definir timeout por etapa com base na latência observada e no SLA do fornecedor.
2. Repetir falhas transitórias com backoff exponencial e jitter, com máximo de tentativas e prazo total.
3. Não repetir automaticamente recusas de negócio, como cartão recusado ou estoque indisponível.
4. Ao estourar o prazo, consultar o status da operação; se o estado permanecer desconhecido, reconciliar e, se necessário, encaminhar para `MANUAL_REVIEW`.
5. Usar um processo de reconciliação para Sagas sem atualização, mensagens perdidas e compensações pendentes.
6. Garantir expiração/renovação da reserva e alinhar o TTL com o tempo máximo esperado da Saga.

## 7. Retries, Circuit Breaker e resiliência

- **Retry:** apenas para erros transitórios (reset de conexão, `429` com política definida, `5xx` recuperáveis); backoff exponencial, jitter e limite de tentativas. O retry de pagamento usa sempre a mesma chave idempotente.
- **Circuit Breaker:** protege o Payment Service e o gateway contra chamadas contínuas quando há falhas ou latência persistentes. A abertura do circuito não define o estado do pagamento; esse estado precisa ser reconciliado.
- **Timeouts e bulkheads:** limitam o tempo de espera e isolam recursos para que uma dependência degradada não esgote threads/conexões dos demais fluxos.
- **Transactional Outbox:** grava a mudança local e a mensagem de saída numa transação única do banco do serviço; um relay publica no broker e pode publicar duplicatas, que serão deduplicadas pelo consumidor.
- **Inbox/Idempotent Consumer:** previnem efeitos repetidos quando o broker entrega novamente uma mensagem após ACK perdido ou falha do consumidor.
- **Reserva com TTL:** libera reservas expiradas por um processo seguro, observando a Saga antes de executar ações que possam conflitar com um pagamento em andamento.

## 8. Dead-letter queues (DLQ)

A DLQ é o destino de mensagens que excederam a política de redelivery ou falharam repetidamente por erro não recuperável. Ela não deve ser tratada como uma fila para ignorar mensagens.

Política proposta:

1. Classificar erro transitório, erro de contrato/schema e erro de negócio; não encaminhar toda recusa de negócio à DLQ.
2. Aplicar retries limitados antes da DLQ, registrando tentativa, causa e timestamps.
3. Incluir metadados de diagnóstico (`messageId`, `sagaId`, tipo/versão, consumer, exceção normalizada e contagem de tentativas), sem incluir credenciais ou dados sensíveis.
4. Alertar quando o volume da DLQ aumentar ou mensagens excederem o tempo máximo de permanência.
5. Reprocessar somente após entender e corrigir a causa; replays devem preservar a chave de idempotência e ser auditados.
6. Ter procedimento operacional para reconciliar a Saga e executar a compensação correspondente. Não editar estado de negócio diretamente sem trilha de auditoria.

## 9. Observabilidade e SLOs

Todos os logs e traces devem carregar `sagaId`, `orderId`, `correlationId` e `messageId`. Dashboards devem observar:

- taxa de Sagas `COMPLETED`, `COMPENSATED` e `MANUAL_REVIEW`;
- duração total da Saga e duração por etapa (p50/p95/p99);
- estoque reservado e idade das reservas;
- pagamento em estado `UNKNOWN` e tempo até reconciliação;
- retries, timeouts, circuit breaker aberto e taxa de erro do gateway;
- tamanho, idade e taxa de crescimento das DLQs;
- compensações pendentes e falhas de entrega/Outbox.

Os limites de alerta e os SLOs devem ser definidos pelo negócio e validados por testes. O sistema deve ter alertas acionáveis e runbooks para pagamentos desconhecidos, compensações presas e crescimento de DLQ.

## 10. Riscos mitigados e riscos residuais

| Risco | Mitigação | Risco residual / observação |
|---|---|---|
| Mensagens duplicadas | Inbox e operações idempotentes | Todas as fronteiras, inclusive gateway, precisam respeitar a chave |
| Evento persistido mas não publicado | Transactional Outbox | Relay pode republicar; consumidor deve deduplicar |
| Timeout com pagamento já aprovado | Estado `PAYMENT_UNKNOWN` e reconciliação | Depende da capacidade de consulta do gateway |
| Reserva de estoque presa | TTL, monitor de reserva e reconciliação com Saga | TTL prematuro pode disputar com uma transação ainda em andamento |
| Saga interrompida após reinício | Estado persistente e retomada por worker | Migração de schema e recuperação precisam ser testadas |
| Mensagem venenosa / schema inválido | Validação, retries limitados e DLQ | Reprocessamento exige correção antes do replay |
| Indisponibilidade do broker | Outbox, retries e alertas | Backlog aumenta latência de conclusão eventual |
| Falha da compensação | Retries idempotentes, estado por compensação e alerta | Pode exigir `MANUAL_REVIEW` em caso irreconciliável |
| Falha/indisponibilidade do gateway | Timeout, Circuit Breaker e consulta de estado | Circuit Breaker não resolve ambiguidade da transação financeira |

## 11. Segurança e auditoria

Não registrar números completos de cartão, CVV, tokens secretos ou credenciais em eventos/logs. O Payment Service deve usar tokenização e os mecanismos de segurança exigidos pelo gateway. Restringir permissões de tópicos, cifrar tráfego e dados sensíveis, definir retenção dos eventos e manter trilha auditável de transições e compensações.

## 12. Plano de validação

Antes de produção, executar testes em ambiente controlado para: resposta de estoque duplicada; mensagem publicada duas vezes; broker indisponível após commit local; timeout do gateway com pagamento aprovado; recusa definitiva; reinício do orquestrador entre etapas; falha temporária de `ReleaseStock`; e mensagem venenosa enviada à DLQ. Validar que as Sagas retomam de estado persistido, não duplicam efeitos financeiros e só chegam a `COMPENSATED` quando todas as compensações exigidas forem confirmadas.

Esta documentação define o comportamento esperado. Resultados de testes, limites operacionais, SLAs e números de configuração devem ser acrescentados após validação no ambiente real.

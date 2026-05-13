---
tags: [php, arquitetura, ddd, cqrs, eventos]
fase: 4
status: draft
---

# Domain Events + CQRS

## Conceito

**Domain Event** é um conceito definido por **Eric Evans** (*DDD*, 2003) e formalizado por **Vaughn Vernon** (*Implementing DDD*, 2013, cap. 8): "*algo de significado para o negócio que aconteceu*". Três propriedades caracterizam:

1. **Imutável** — é um fato passado; fatos não mudam.
2. **Nomeado no passado** — `OrderPaid`, `InventoryReserved`, `SubscriptionCancelled`. Não `PayOrder` (isso é command).
3. **Carrega informação suficiente** para o consumidor reagir, sem precisar voltar a consultar o agregado.

O padrão operacional canônico tem três passos:

```
   ┌────────────────────────────────────────────────────────────────┐
   │                                                                │
   │   1. Aggregate Root COLETA eventos durante mutação             │
   │      (não despacha — só guarda em $pendingEvents)              │
   │                                                                │
   │   2. Use Case PERSISTE o agregado                              │
   │      (na mesma transação, grava eventos em outbox table)       │
   │                                                                │
   │   3. Worker LÊ outbox e DESPACHA eventos                       │
   │      (at-least-once, com ack ao consumir)                      │
   │                                                                │
   └────────────────────────────────────────────────────────────────┘
```

O **Transactional Outbox** existe para resolver um problema fundamental: se você comita no banco e **depois** publica no bus (RabbitMQ, SNS), a publicação pode falhar e o evento se perde. Se você publica **antes** do commit, o handler pode reagir a algo que nunca foi persistido. A solução é **gravar o evento na mesma transação do agregado, em uma tabela `outbox`**, e ter um worker separado que lê dessa tabela e despacha para o bus real. Garante *at-least-once delivery* (handlers precisam ser idempotentes).

```
       BAD (lost event):                BAD (phantom event):
       commit() ───── ✓                 publish() ───── ✓
       publish() ──── ✗                 commit() ───── ✗
       evento perdido                   handler reage a estado inexistente

       GOOD (outbox):
       BEGIN
         save aggregate
         insert into outbox(event)
       COMMIT
         ───── ✓
       (worker assíncrono lê outbox e despacha; idempotência no consumidor)
```

**Síncrono vs Assíncrono**: handlers síncronos rodam na mesma transação do produtor — falham juntos, comitam juntos. Servem para *reações dentro do mesmo bounded context* (ex: atualizar um *read model* local). Handlers assíncronos rodam depois, em worker separado — servem para *integração entre contextos* e *side effects pesados* (e-mail, webhook, indexação). A regra prática: dentro de contexto, síncrono; entre contextos, assíncrono via outbox + fila.

**Event-carried state transfer** (Martin Fowler): o evento carrega **dados suficientes** para o consumidor não precisar consultar o produtor de volta. `OrderPaid` carrega `OrderId`, `Total`, `CustomerId`, `PaidAt` — o consumidor de email tem o necessário sem chamar `OrderRepository`. Reduz acoplamento e elimina a dependência circular de leitura.

**Domain Events ≠ Event Sourcing**. São coisas distintas que muita gente confunde:

| Aspecto       | Domain Events                       | Event Sourcing                              |
|---------------|-------------------------------------|---------------------------------------------|
| Persistência  | Agregado salvo em tabela "estado"   | Agregado é **derivado** de stream de eventos |
| Eventos       | Subproduto da mutação               | É a verdade — não há "estado" salvo         |
| Reconstrução  | Carrega linha do banco              | Replaying do stream                         |
| Custo         | Baixo                               | Alto (versionamento, snapshot, debug)       |
| Quando usar   | Quase sempre que faz DDD            | Raramente — auditoria forte, financeiro     |

Você pode (e na maioria das vezes deve) ter Domain Events **sem** Event Sourcing.

## CQRS

**CQRS** (*Command/Query Responsibility Segregation*, Greg Young, 2010, baseado em Bertrand Meyer) é a separação entre o **modelo de escrita** (agregado, otimizado para invariantes) e o **modelo de leitura** (*read model*, otimizado para consulta). A motivação: o que é bom para garantir invariantes (agregado normalizado, transacional) é péssimo para listar/buscar (precisa de joins, projeções, agregações). Em vez de torcer um modelo único, **mantenha dois**.

```
      Write side                              Read side
      ┌──────────────┐                        ┌─────────────────┐
      │ Aggregate    │                        │ Read Model      │
      │ Repository   │                        │ (denormalizado) │
      │ (write DB)   │                        │ (read DB / view)│
      └──────┬───────┘                        └────────▲────────┘
             │                                         │
             ▼                                         │
      Domain Event ─────── handler ───────► projeta no read model
```

CQRS **não exige**: filas, banco separado, event sourcing. A forma mínima é:

- *Command Handler* escreve no modelo de domínio.
- *Query Handler* lê de uma projeção (pode ser uma `VIEW` SQL, uma tabela denormalizada, ou até a mesma tabela com SELECT enxuto).

CQRS **paga a conta** quando há divergência clara entre necessidades de escrita e leitura. Não é "obrigatório em DDD".

**PSR-14 vs Domain Bus**: PSR-14 é um *Event Dispatcher* genérico para eventos de aplicação (úteis para frameworks, plugins). Para Domain Events, costuma valer a pena ter um *bus específico do domínio*, tipado, com semântica de "evento de negócio" — mais legível e mais checável estaticamente.

## Por que importa

1. **Reatividade sem acoplamento.** Sem eventos, `pay()` chama explicitamente `inventory->reserve()`, `mailer->send()`, `analytics->track()`. O Use Case cresce, conhece tudo, depende de tudo. Com eventos, `pay()` emite `OrderPaid`; cada side effect é um handler isolado. Adicionar "também envia SMS" = adicionar um handler novo, zero alteração no fluxo principal.

2. **Auditoria gratuita.** Stream de eventos é o histórico do negócio. "Quando esse pedido foi pago? Por qual gateway? Quem cancelou?". Sem eventos, você reconstrói via logs e snapshots; com eventos, é só ler.

3. **Integração entre contextos.** Bounded Context A não conhece B; mas B reage ao evento de A. Acoplamento via mensagem, não via chamada de método.

4. **Read models otimizados.** Consultas pesadas (relatório, dashboard, busca facetada) deixam de afetar a tabela transacional. Read model fica em índice apropriado (Postgres com índice GIN, Elasticsearch, materialized view).

5. **Resiliência.** Falha no envio de e-mail (handler) não falha o pagamento (transação do agregado). Outbox + retry resolve a entrega; o domínio segue.

O custo: complexidade operacional. Outbox table, worker, monitoramento de lag, idempotência nos handlers. Em sistemas pequenos não paga.

## Exemplo de código PHP

### 1. Domain Event como VO imutável

```php
<?php
declare(strict_types=1);

namespace App\Domain\Order\Events;

use App\Domain\Order\OrderId;
use App\Domain\Customer\CustomerId;
use App\Domain\Shared\Money;

final readonly class OrderPaid
{
    public function __construct(
        public OrderId $orderId,
        public CustomerId $customerId,
        public Money $total,
        public string $providerTransactionId,
        public \DateTimeImmutable $paidAt,
    ) {}
}
```

### 2. Aggregate Root coleta eventos durante mutação

```php
<?php
declare(strict_types=1);

namespace App\Domain\Order;

use App\Domain\Order\Events\OrderPaid;
use App\Domain\Shared\Money;

final class Order
{
    /** @var list<object> */
    private array $pendingEvents = [];

    public function pay(string $providerTxId, \DateTimeImmutable $now): void
    {
        if (!$this->status->canTransitionTo(OrderStatus::Paid)) {
            throw new InvalidStateTransition($this->status, OrderStatus::Paid);
        }

        $this->status = OrderStatus::Paid;

        // Evento coletado, NÃO despachado aqui
        $this->pendingEvents[] = new OrderPaid(
            $this->id, $this->customer, $this->total(), $providerTxId, $now,
        );
    }

    /** @return list<object> */
    public function pullEvents(): array
    {
        $events = $this->pendingEvents;
        $this->pendingEvents = [];
        return $events;
    }
}
```

### 3. Outbox: persistência atômica de estado + eventos

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Outbox;

final readonly class OutboxAwareTransaction
{
    public function __construct(
        private \PDO $pdo,
        private OutboxStore $outbox,
    ) {}

    /**
     * @template T
     * @param callable():array{0:T,1:list<object>} $work  retorna [resultado, eventos]
     * @return T
     */
    public function run(callable $work): mixed
    {
        $this->pdo->beginTransaction();
        try {
            [$result, $events] = $work();

            // Grava eventos na MESMA transação do estado
            foreach ($events as $event) {
                $this->outbox->store($event);
            }

            $this->pdo->commit();
            return $result;
        } catch (\Throwable $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }
}
```

```php
<?php
declare(strict_types=1);

// Worker separado lê outbox e despacha
final readonly class OutboxRelayWorker
{
    public function __construct(
        private OutboxStore $outbox,
        private EventDispatcher $bus,
    ) {}

    public function tick(int $batchSize = 100): void
    {
        foreach ($this->outbox->pending($batchSize) as $row) {
            try {
                $this->bus->dispatch($row->event);
                $this->outbox->markDispatched($row->id);
            } catch (\Throwable $e) {
                $this->outbox->markFailed($row->id, $e->getMessage());
                // retry com backoff exponencial
            }
        }
    }
}
```

### 4. Handler de evento (consumidor)

```php
<?php
declare(strict_types=1);

namespace App\Application\Notification;

use App\Domain\Order\Events\OrderPaid;

final readonly class SendReceiptOnOrderPaid
{
    public function __construct(
        private CustomerRepository $customers,
        private Mailer $mailer,
        private IdempotencyStore $idempotency,
    ) {}

    public function __invoke(OrderPaid $event): void
    {
        // Idempotência: handler pode ser chamado mais de uma vez (at-least-once)
        $key = "receipt:{$event->orderId->value}";
        if ($this->idempotency->has($key)) {
            return;
        }

        $customer = $this->customers->ofId($event->customerId);
        $this->mailer->sendReceipt($customer->email, $event);

        $this->idempotency->put($key, true);
    }
}
```

### 5. CQRS — Read Model puro

```php
<?php
declare(strict_types=1);

namespace App\Application\Order\Read;

// Read Model: DTO, sem comportamento, denormalizado para a query
final readonly class OrderListItem
{
    public function __construct(
        public string $id,
        public string $customerName,
        public int $totalCents,
        public string $currency,
        public string $status,
        public \DateTimeImmutable $createdAt,
    ) {}
}

interface OrderListQuery
{
    /** @return list<OrderListItem> */
    public function recentByCustomer(string $customerId, int $limit = 20): array;
}
```

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\ReadModel;

use App\Application\Order\Read\{OrderListItem, OrderListQuery};

// Implementação otimizada: SELECT com join, sem carregar agregado
final readonly class PdoOrderListQuery implements OrderListQuery
{
    public function __construct(private \PDO $pdo) {}

    public function recentByCustomer(string $customerId, int $limit = 20): array
    {
        $stmt = $this->pdo->prepare(
            'SELECT o.id, c.name AS customer_name, o.total_cents,
                    o.currency, o.status, o.created_at
             FROM orders o
             JOIN customers c ON c.id = o.customer_id
             WHERE o.customer_id = :customer
             ORDER BY o.created_at DESC
             LIMIT :limit'
        );
        $stmt->execute(['customer' => $customerId, 'limit' => $limit]);

        return array_map(
            fn (array $row) => new OrderListItem(
                $row['id'],
                $row['customer_name'],
                (int) $row['total_cents'],
                $row['currency'],
                $row['status'],
                new \DateTimeImmutable($row['created_at']),
            ),
            $stmt->fetchAll(\PDO::FETCH_ASSOC),
        );
    }
}
```

### 6. Projeção alimentada por evento (CQRS + Domain Event)

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\ReadModel;

use App\Domain\Order\Events\OrderPaid;

// Handler que atualiza o read model quando o evento chega
final readonly class UpdateOrderListOnPaid
{
    public function __construct(private \PDO $pdo) {}

    public function __invoke(OrderPaid $event): void
    {
        $this->pdo->prepare('UPDATE orders SET status = :status WHERE id = :id')
            ->execute([
                'status' => 'paid',
                'id'     => $event->orderId->value,
            ]);
    }
}
```

## Quando usar / quando não usar

- **Domain Events**: use sempre que um fato do negócio deva acionar ≥ 1 reação independente. Praticamente todo sistema DDD se beneficia.
- **Outbox**: use quando perder evento é inaceitável (pagamento, fiscal, integração entre serviços). Boilerplate justificado.
- **Eventos síncronos in-process**: bom para atualizar read model local na mesma transação. Simples e suficiente em muitos casos.
- **Eventos assíncronos (fila)**: use para integração entre bounded contexts ou side effects pesados/lentos.
- **CQRS**: use quando consultas pesadas degradam o modelo transacional, ou quando leitura precisa de joins/projeções incompatíveis com escrita. **Não** é obrigatório em DDD.
- **Não use Event Sourcing** sem necessidade clara (auditoria regulatória, financeiro com replay). ES é caro de operar, debug é difícil, versionamento de evento é tema delicado.
- **Não use CQRS** em CRUD trivial. Read = SELECT direto basta.
- **PSR-14** serve para eventos de aplicação/framework; para Domain Events vale a pena bus próprio tipado.

## Armadilhas comuns

- **Despachar evento antes de salvar.** Handler reage a algo que pode rollback. Sempre persista primeiro (com outbox na mesma transação) e despache depois.
- **Handler síncrono crítico que falha silenciosamente.** `SendEmailOnOrderPaid` síncrono falha — pagamento dá rollback. Use outbox + worker + retry, e mantenha o domínio independente do side effect.
- **Read model defasado e ninguém percebe.** Handler de projeção falhou; lista mostra estado errado. Monitore lag entre eventos e projeção. Reprojeção (rebuild a partir do stream) deve ser viável.
- **Tratar Read Model como entidade.** Read Model é DTO. Sem invariantes, sem comportamento. Se você está chamando método em read model, errou a camada.
- **Confundir Domain Event com Integration Event.** Domain Event = vocabulário interno do contexto; rico em tipos do domínio. Integration Event = forma de saída entre contextos; primitivos, versionado, contrato estável. Pode ser o mesmo objeto, mas costuma valer separar.
- **Event-carried state transfer com payload gigante.** Carregar o agregado inteiro no evento polui o bus e versionamento. Carregue o mínimo necessário.
- **Handler não idempotente em outbox.** Outbox garante at-least-once — handler pode rodar 2+ vezes. Use idempotency key (event id) e store de "já processei".
- **CQRS sem real necessidade.** Read model duplicado para uma listagem que cabe num SELECT simples. Cerimônia sem retorno. Adote CQRS quando dor de leitura aparece, não preventivamente.
- **Confundir CQRS com Event Sourcing.** São independentes. CQRS sem ES é comum e útil. ES sem CQRS é raro mas existe.
- **PSR-14 para tudo.** PSR-14 é genérico, eventos não são tipados pelo contrato. Para Domain Events tipados, bus específico do projeto traz clareza.

## Links relacionados

- [[Aggregates]]
- [[Use-Cases]]
- [[Camadas-DDD]]
- [[Hexagonal]]
- [[Padroes/Observer]]
- [[Performance/Filas]]

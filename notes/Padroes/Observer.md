---
tags: [php, padroes, observer, eventos]
fase: 3
status: draft
---

# Observer / Event Dispatcher

## Conceito

Observer é um padrão comportamental do GoF: um **sujeito** (`Subject`) notifica **observadores** (`Observer`) quando algo acontece, sem conhecê-los individualmente. O sujeito mantém uma lista de subscribers; ao mudar estado, percorre a lista chamando `update($event)`.

```
        +--------------+ notify  +-------------+
        |   Subject    +-------->| Observer 1  |
        +-------+------+         +-------------+
                |    \           +-------------+
                |     +--------->| Observer 2  |
                |                +-------------+
                |                +-------------+
                +--------------->| Observer 3  |
                                 +-------------+
```

Em PHP moderno, o Observer clássico cedeu espaço a duas materializações mais robustas:

1. **PSR-14 Event Dispatcher**: padrão de interoperabilidade. Define `EventDispatcherInterface::dispatch(object $event)`, `ListenerProviderInterface::getListenersForEvent(object $event): iterable`, e `StoppableEventInterface::isPropagationStopped(): bool`. Listeners são callables registrados por tipo de evento.
2. **Domain Events**: o agregado **acumula eventos** internamente durante operações de domínio (`$order->pay()` adiciona `OrderPaid` a uma lista interna). Após persistir, um bus despacha esses eventos. Característica chave: eventos são parte da linguagem ubíqua, expressos como VOs no passado (`OrderPaid`, `PaymentFailed`).

Comparado a padrões irmãos:

| Padrão | Foco | Diferença chave |
|---|---|---|
| **Observer (clássico)** | Sujeito conhece a lista de observers | Acoplamento direto sujeito ↔ lista |
| **Event Dispatcher** | Bus intermediário gerencia subscribers | Sujeito **não conhece** observers; só publica |
| **Mediator** | Objeto central coordena interações | Mediator decide o **fluxo**; Dispatcher só entrega |
| **Pub/Sub (mensageria)** | Publisher e subscriber em processos diferentes | Assíncrono, fila persistente; Dispatcher PSR-14 é in-process |
| **Command Bus** | Despacha comando para **único** handler | Command tem um destinatário; evento, N |

**Eventos síncronos vs assíncronos**:

- **Síncrono**: listener executa no mesmo request/transaction. Simples; risco de latência e cascata de falha.
- **Assíncrono**: dispatcher enfileira (Redis, RabbitMQ, SQS); worker processa fora. Resiliência e desacoplamento temporal; complexidade de infra e de rastreio.

Híbrido idiomático: listeners **rápidos e críticos** (auditoria local, invalidação de cache) síncronos; **lentos e externos** (email, webhook, sync com ERP) enfileirados.

## Por que importa

Observer/Events desacoplam "quem decide que algo aconteceu" de "quem reage". Permitem que `Order::pay()` emita `OrderPaid` e três bounded contexts diferentes (notificação, BI, fiscal) reajam — sem o domínio saber deles.

Cenários concretos:

- **Pagamento confirmado** → enviar email + atualizar dashboard + emitir nota fiscal + notificar fulfillment. Em vez do use case `PayOrder` conhecer 4 colaboradores, dispara um evento; 4 listeners reagem.
- **Usuário registrado** → criar perfil padrão + enviar boas-vindas + adicionar em CRM. Mesmo princípio.
- **Webhook recebido** → traduzir para evento de domínio, processar, despachar consequências.
- **Auditoria automática**: listener escuta todos os eventos de mudança e grava em `audit_log` — código de auditoria zero no domínio.

Quando **não** aplicar:

- Consumidor único e síncrono — chamada direta é mais clara e debugável.
- Fluxo crítico transacional com rollback — gerenciar transação distribuída via eventos in-memory é frágil.
- Quando o "evento" é na verdade um comando ("envie email") — use [[Strategy]] ou injeção direta.

## Exemplo de código PHP

### 1) Eventos como VOs imutáveis

```php
<?php
declare(strict_types=1);

// Eventos descrevem fatos do passado. Nome no particípio. Readonly.
final readonly class OrderPaid
{
    public function __construct(
        public OrderId $orderId,
        public CustomerId $customerId,
        public Money $amount,
        public \DateTimeImmutable $paidAt,
    ) {}
}

final readonly class OrderShipped
{
    public function __construct(
        public OrderId $orderId,
        public TrackingCode $tracking,
        public \DateTimeImmutable $shippedAt,
    ) {}
}
```

### 2) PSR-14 Listener Provider e Dispatcher

```php
<?php
declare(strict_types=1);

use Psr\EventDispatcher\EventDispatcherInterface;
use Psr\EventDispatcher\ListenerProviderInterface;
use Psr\EventDispatcher\StoppableEventInterface;

final class InMemoryListenerProvider implements ListenerProviderInterface
{
    /** @var array<class-string, list<callable>> */
    private array $listeners = [];

    /** @param class-string $eventClass */
    public function on(string $eventClass, callable $listener): void
    {
        $this->listeners[$eventClass][] = $listener;
    }

    public function getListenersForEvent(object $event): iterable
    {
        $eventClass = $event::class;
        // Listeners diretos
        yield from $this->listeners[$eventClass] ?? [];
        // Listeners de interfaces/pais — útil para "listenerOnAnyEvent"
        foreach (class_implements($event) ?: [] as $iface) {
            yield from $this->listeners[$iface] ?? [];
        }
    }
}

final readonly class EventDispatcher implements EventDispatcherInterface
{
    public function __construct(private ListenerProviderInterface $provider) {}

    public function dispatch(object $event): object
    {
        foreach ($this->provider->getListenersForEvent($event) as $listener) {
            if ($event instanceof StoppableEventInterface
                && $event->isPropagationStopped()
            ) {
                break;
            }
            $listener($event);
        }
        return $event;
    }
}
```

### 3) Registro e despacho

```php
<?php
declare(strict_types=1);

$provider = new InMemoryListenerProvider();

// Listeners como Closures, callables ou classes invocáveis.
$provider->on(OrderPaid::class, fn (OrderPaid $e) => $mailer->confirm($e));
$provider->on(OrderPaid::class, fn (OrderPaid $e) => $analytics->track($e));
$provider->on(OrderPaid::class, $invoiceIssuer);  // classe com __invoke

$dispatcher = new EventDispatcher($provider);

// Domínio publica fato; não conhece listeners.
$dispatcher->dispatch(new OrderPaid(
    orderId: $order->id,
    customerId: $order->customerId,
    amount: $order->total,
    paidAt: new \DateTimeImmutable(),
));
```

### 4) Agregado acumulando eventos (Domain Events)

```php
<?php
declare(strict_types=1);

abstract class AggregateRoot
{
    /** @var list<object> */
    private array $pendingEvents = [];

    protected function recordEvent(object $event): void
    {
        $this->pendingEvents[] = $event;
    }

    /** @return list<object> */
    public function pullEvents(): array
    {
        $events = $this->pendingEvents;
        $this->pendingEvents = [];
        return $events;
    }
}

final class Order extends AggregateRoot
{
    public function pay(Money $amount): void
    {
        if ($this->status !== OrderStatus::Pending) {
            throw new \DomainException('order not pending');
        }
        $this->status = OrderStatus::Paid;
        $this->recordEvent(new OrderPaid(
            orderId: $this->id,
            customerId: $this->customerId,
            amount: $amount,
            paidAt: new \DateTimeImmutable(),
        ));
    }
}

// Use case despacha após persistir.
final readonly class PayOrder
{
    public function __construct(
        private OrderRepository $orders,
        private EventDispatcherInterface $events,
    ) {}

    public function __invoke(OrderId $id, Money $amount): void
    {
        $order = $this->orders->ofId($id) ?? throw new \DomainException('not found');
        $order->pay($amount);
        $this->orders->save($order);
        foreach ($order->pullEvents() as $event) {
            $this->events->dispatch($event);
        }
    }
}
```

### 5) Evento stoppable

```php
<?php
declare(strict_types=1);

use Psr\EventDispatcher\StoppableEventInterface;

final class BeforeOrderShipped implements StoppableEventInterface
{
    private bool $stopped = false;

    public function __construct(
        public readonly Order $order,
    ) {}

    public function stop(): void
    {
        $this->stopped = true;
    }

    public function isPropagationStopped(): bool
    {
        return $this->stopped;
    }
}

// Listener que cancela o envio se há restrição.
$provider->on(BeforeOrderShipped::class, function (BeforeOrderShipped $e): void {
    if ($e->order->hasBackorder()) {
        $e->stop();  // demais listeners não rodam
    }
});
```

### 6) Listener assíncrono (enfileirado)

```php
<?php
declare(strict_types=1);

// Listener registrado é leve: só enfileira para worker processar.
$provider->on(OrderPaid::class, function (OrderPaid $e) use ($queue): void {
    $queue->push('notifications', [
        'type' => 'order_paid',
        'order_id' => $e->orderId->toString(),
        'amount_cents' => $e->amount->cents,
    ]);
});
```

## Quando usar / quando não usar

- **Usar Domain Events** entre bounded contexts — desacoplamento real entre módulos do mesmo sistema.
- **Usar Event Dispatcher** para hooks de aplicação: auditoria, métricas, webhooks de saída, side effects opcionais.
- **Usar assíncrono** quando o listener faz I/O lento (email, webhook, sync externo) e a falha dele não deve quebrar o fluxo principal.
- **Usar síncrono** quando o listener é rápido, local, e seu sucesso é parte da consistência (atualizar cache de leitura, marcar idempotência).
- **Não usar** quando o "consumidor" é único e síncrono — chamada direta é mais clara, mais rápida, mais debugável.
- **Não usar** para fluxo transacional crítico in-memory — se o handler falha, fica difícil rastrear e compensar. Para isso, fila persistente com retry.
- **Não usar** quando o evento na verdade é um comando ("envie email") — use [[Strategy]] ou injeção direta.

## Armadilhas comuns

- **Listener síncrono lento em transação HTTP**: handler que faz HTTP externo estoura SLA. Prefira enfileirar.
- **Ordem de execução de listeners não garantida**: PSR-14 não promete ordem entre listeners do mesmo evento. Não dependa. Se ordem importa, considere coordenador explícito ou Pipeline.
- **Loop de eventos**: handler que dispara novo evento que aciona o mesmo handler. Detecte na hora; use guarda por correlation id ou flag de profundidade.
- **Despachar antes de persistir**: agregado emite evento, listener atualiza outro contexto, persistência falha — outro contexto fica inconsistente. Padrão: persista primeiro, depois despache (ou use Outbox Pattern para garantir).
- **Listener mutando o agregado original**: listener recebe evento e altera entidade — vira código em ordem implícita, difícil de rastrear. Listeners reagem com **side effects** ou disparam novos comandos, não mutam o agregado.
- **Evento como command disfarçado**: `EmailSendRequested` não é evento, é comando. Eventos são **fatos do passado**, não pedidos.
- **Engolir exceção no listener**: listener que captura silenciosamente faz auditoria sumir. Logue e re-lance, ou tenha estratégia explícita de retry/dead-letter.
- **Faltar correlation/causation id**: sem ids para correlacionar evento → causa → consequência, rastreio em produção vira detetive sem pistas.

## Links relacionados

- [[Decorator]]
- [[Strategy]]
- [[Repository]]
- [[Specification]]
- [[Padroes/SOLID]]
- [[Arquitetura/Domain-Events-CQRS]]
- [[Performance/Filas]]

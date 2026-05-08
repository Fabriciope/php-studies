---
tags: [php, padroes, observer, eventos]
fase: 3
status: stub
---

# Observer / Event Dispatcher

## Conceito

Um sujeito (`Subject`) notifica observers quando algo acontece, sem conhecê-los individualmente. Em PHP moderno, materializa-se como **PSR-14 Event Dispatcher** ou Domain Events: o agregado guarda eventos, um bus os entrega aos handlers registrados.

## Por que importa

Desacopla "quem decide que algo aconteceu" de "quem reage a isso". Permite que `Pedido::pay()` emita `OrderPaid` e três bounded contexts diferentes (notificação, BI, fiscal) reajam — sem o domínio saber deles.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// 1) eventos como VOs imutáveis
final readonly class OrderPaid {
    public function __construct(
        public OrderId $id,
        public Money $amount,
        public \DateTimeImmutable $at,
    ) {}
}

// 2) PSR-14: ListenerProvider devolve listeners por tipo
interface ListenerProvider {
    /** @return iterable<callable> */
    public function getListenersForEvent(object $event): iterable;
}

final class InMemoryListenerProvider implements ListenerProvider {
    /** @var array<class-string, list<callable>> */
    private array $map = [];

    public function on(string $eventClass, callable $listener): void {
        $this->map[$eventClass][] = $listener;
    }

    public function getListenersForEvent(object $event): iterable {
        return $this->map[$event::class] ?? [];
    }
}

// 3) dispatcher
final readonly class EventDispatcher {
    public function __construct(private ListenerProvider $provider) {}
    public function dispatch(object $event): void {
        foreach ($this->provider->getListenersForEvent($event) as $listener) {
            $listener($event);
        }
    }
}

// uso
$provider = new InMemoryListenerProvider();
$provider->on(OrderPaid::class, fn (OrderPaid $e) => $mailer->confirm($e));
$provider->on(OrderPaid::class, fn (OrderPaid $e) => $analytics->track($e));
$dispatcher = new EventDispatcher($provider);
$dispatcher->dispatch(new OrderPaid($id, $total, $now));
```

## Quando usar / quando não usar

- **Domain events** entre bounded contexts — desacoplamento real.
- **Hooks de aplicação** (auditoria, métricas, webhooks).
- **Não usar** quando o "consumidor" é único e síncrono — chamada direta é mais clara.
- **Não usar** para fluxo de transação crítico em-memória — se o handler falha, fica difícil rastrear. Para isso, fila persistente.

## Armadilhas comuns

- Listener síncrono lento dentro de transação HTTP estoura SLA. Prefira despachar pra fila.
- Ordem de execução de listeners não é garantida — não dependa.
- Loop de eventos: handler que dispara novo evento que aciona o mesmo handler. Detecte na hora.

## Links relacionados

- [[Arquitetura/Domain-Events-CQRS]]
- [[Performance/Filas]]
- [[Specification]]

---
tags: [php, oop, php8]
fase: 2
status: stub
---

# Constructor Promotion + First-Class Callable

## Conceito

**Constructor promotion** (PHP 8.0): declarar e atribuir propriedades direto na assinatura do construtor. **First-class callable syntax** (PHP 8.1): `func(...)` ou `$obj->method(...)` produzem um `Closure` referenciando aquele método/função.

## Por que importa

Promotion remove o ritual `__construct + $this->x = $x` que dominava 30% de qualquer classe PHP. First-class callable acaba com `[$obj, 'method']` e `'strlen'` como string mágica — vira tipo de primeira classe testável e estaticamente verificável.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// promotion: declaração + atribuição em uma linha
final readonly class CreateOrderHandler {
    public function __construct(
        private OrderRepository $orders,
        private EventBus $bus,
        private Clock $clock,
    ) {}

    public function handle(CreateOrder $cmd): OrderId {
        $order = Order::create($cmd, $this->clock->now());
        $this->orders->save($order);
        foreach ($order->pullEvents() as $event) {
            $this->bus->dispatch($event);
        }
        return $order->id;
    }
}

// first-class callable
$mapper = strtoupper(...);            // Closure
$names  = array_map($mapper, $names);

$handler = $cmdHandler->handle(...);  // método como Closure
$pipeline->add($handler);
```

## Quando usar / quando não usar

- **Promotion sempre** que o construtor só atribui — é o caso 90% das vezes em handlers, services, VOs.
- **Combinar com `readonly`** para o conjunto canônico de classe imutável injetada.
- **Não usar promotion** se houver lógica não trivial entre os parâmetros (validação cruzada) — use construtor explícito para clareza.
- **First-class callable** como substituto de strings/array calls sempre que o linter for ajudar.

## Armadilhas comuns

- Promotion + `readonly` exige PHP ≥ 8.1.
- Você ainda pode adicionar corpo no `__construct` — lógica adicional roda **depois** das atribuições da assinatura.
- `$obj->method(...)` cria Closure **vinculado** a `$obj`. Mantém referência — atenção a leaks em listeners de longa vida.

## Links relacionados

- [[Readonly-Classes]]
- [[Asymmetric-Visibility]]
- [[Padroes/SOLID]]

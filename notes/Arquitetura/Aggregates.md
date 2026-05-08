---
tags: [php, arquitetura, ddd, aggregate]
fase: 4
status: stub
---

# Aggregates

## Conceito

Cluster de entidades e VOs tratado como **uma unidade de consistência**. Tem uma **raiz** (Aggregate Root) — o único ponto pelo qual o mundo externo manipula o cluster. Invariantes do cluster são protegidos pela raiz.

Regra de ouro: **uma transação = um agregado**. Mudou A e B no mesmo commit? São o mesmo agregado, ou está errado.

## Por que importa

Sem agregados, qualquer parte do sistema pode mutar qualquer entidade — invariantes ficam dispersos, validações duplicadas, race conditions inevitáveis. O agregado **concentra** as regras e força que mudanças passem pela porta certa.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

final class Order {
    /** @var list<OrderItem> */
    private array $items = [];
    private array $events = [];

    public function __construct(
        public readonly OrderId $id,
        public readonly CustomerId $customer,
        public private(set) OrderStatus $status,
    ) {}

    // mundo externo NÃO mexe em items diretamente — passa pela raiz
    public function addItem(ProductId $product, int $qty, Money $unitPrice): void {
        if ($this->status !== OrderStatus::Pending) {
            throw new DomainException('cannot modify confirmed order');
        }
        if ($qty <= 0) {
            throw new InvalidArgumentException('qty > 0');
        }
        $this->items[] = new OrderItem($product, $qty, $unitPrice);
    }

    public function markPaid(string $providerId): void {
        if (!$this->status->canTransitionTo(OrderStatus::Paid)) {
            throw new InvalidStateTransition();
        }
        $this->status = OrderStatus::Paid;
        $this->events[] = new OrderPaid($this->id, $this->total(), new \DateTimeImmutable());
    }

    public function total(): Money {
        return array_reduce(
            $this->items,
            fn (Money $acc, OrderItem $i) => $acc->add($i->subtotal()),
            new Money(0, 'BRL'),
        );
    }

    /** @return list<object> */
    public function pullEvents(): array {
        $e = $this->events; $this->events = []; return $e;
    }
}
```

## Quando usar / quando não usar

- **Sempre** quando há invariantes entre múltiplas entidades (ordem + itens + cupons).
- **Mantenha pequeno** — agregados gigantes (`Customer` com tudo) viram lock contention.
- **Referência por ID** entre agregados — nunca por objeto. `Order` referencia `CustomerId`, não `Customer`.

## Armadilhas comuns

- Carregar agregado pela metade ("só quero o status"). Repository devolve raiz inteira; se está pesado, repense fronteira ou use **read model**.
- Invariante validado fora do agregado (no Use Case, no controller). Volta pra raiz.
- Mutar items via `getItems()` retornando array por referência. Devolva `iterable` ou clone.

## Links relacionados

- [[Value-Objects]]
- [[Use-Cases]]
- [[Domain-Events-CQRS]]

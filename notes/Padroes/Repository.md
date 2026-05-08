---
tags: [php, padroes, persistencia]
fase: 3
status: stub
---

# Repository

## Conceito

Abstração tipo "coleção em memória" sobre persistência. O domínio fala `$orders->save($order)` e `$orders->ofId($id)`; quem implementa pode ser MySQL, in-memory, arquivo, API. O domínio não sabe nem importa.

## Por que importa

Sem Repository, regras de negócio dependem de SQL ou ORM — testes ficam lentos, troca de banco vira projeto, e detalhes de persistência (`$query->whereNull(...)`) vazam para use cases. Repository é a fronteira entre **modelo** e **storage**.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

interface OrderRepository {
    public function save(Order $order): void;
    public function ofId(OrderId $id): ?Order;
    /** @return iterable<Order> */
    public function pendingOlderThan(\DateTimeImmutable $cutoff): iterable;
}

// adapter PDO
final readonly class PdoOrderRepository implements OrderRepository {
    public function __construct(private \PDO $pdo) {}

    public function save(Order $order): void {
        $stmt = $this->pdo->prepare(
            'INSERT INTO orders (id, status, total_cents) VALUES (?,?,?)
             ON DUPLICATE KEY UPDATE status=VALUES(status), total_cents=VALUES(total_cents)'
        );
        $stmt->execute([
            $order->id->toString(),
            $order->status->value,
            $order->total->cents,
        ]);
    }
    /* ... */
}

// adapter in-memory para testes
final class InMemoryOrderRepository implements OrderRepository {
    /** @var array<string, Order> */
    private array $store = [];
    public function save(Order $order): void { $this->store[$order->id->toString()] = $order; }
    public function ofId(OrderId $id): ?Order { return $this->store[$id->toString()] ?? null; }
    public function pendingOlderThan(\DateTimeImmutable $cutoff): iterable { /* ... */ }
}
```

## Quando usar / quando não usar

- **Sempre** entre domínio e persistência em arquitetura em camadas/hexagonal.
- **Não usar** em scripts CRUD simples sem regras de negócio — ali o ORM já é o modelo, repository é cerimônia vazia.
- **Não vire DAO** com método para cada query — Repository expressa **conceitos do domínio** ("pedidos pendentes há mais de X"), não SQL.

## Armadilhas comuns

- `findBy(['status' => 'paid'])` — interface de repository que aceita mapa genérico vaza detalhe relacional. Prefira métodos nomeados de domínio.
- Repository expondo Query Builder do ORM — quebra a abstração.
- Combinar com [[Specification]] quando o número de "find*" explode.

## Links relacionados

- [[Specification]]
- [[Decorator]]
- [[Arquitetura/Hexagonal]]

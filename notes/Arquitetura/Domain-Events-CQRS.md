---
tags: [php, arquitetura, ddd, cqrs, eventos]
fase: 4
status: stub
---

# Domain Events + CQRS

## Conceito

**Domain Event**: fato passado, imutável, modelado como VO (`OrderPaid`, `InventoryReserved`). Emergido pelo agregado, despachado após `save`.

**CQRS** (Command/Query Responsibility Segregation): separar o **modelo de escrita** (agregado, transacional) do **modelo de leitura** (read model, otimizado por consulta). Não exige fila, ES ou DB separado — só modelos diferentes.

## Por que importa

Domain Events tornam o sistema **reativo sem acoplar bounded contexts**. CQRS resolve a tensão entre "agregado bom para escrever" e "denormalizado bom para ler" — em vez de torcer um para servir os dois, modele cada um para o que faz.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// Domain Event: imutável, no passado, no domínio
namespace App\Domain\Order\Events;

final readonly class OrderPaid {
    public function __construct(
        public OrderId $id,
        public Money $total,
        public \DateTimeImmutable $paidAt,
    ) {}
}

// despacho ocorre APÓS persistir (outbox pattern para garantia)
final class TransactionalDispatcher {
    public function __construct(
        private \PDO $pdo,
        private EventDispatcher $bus,
    ) {}

    public function run(callable $work, array $events): void {
        $this->pdo->beginTransaction();
        try {
            $work();
            // outbox: gravar eventos na MESMA transação
            $this->saveOutbox($events);
            $this->pdo->commit();
        } catch (\Throwable $e) {
            $this->pdo->rollBack();
            throw $e;
        }
        // worker separado lê outbox e despacha — at-least-once garantido
    }
}

// CQRS — read model é VO, não agregado
final readonly class OrderListItem {
    public function __construct(
        public string $id,
        public string $customerName,
        public int $totalCents,
        public string $status,
        public \DateTimeImmutable $createdAt,
    ) {}
}

interface OrderQuery {
    /** @return list<OrderListItem> */
    public function recentByCustomer(CustomerId $customer, int $limit = 20): array;
}
```

## Quando usar / quando não usar

- **Domain Events** — sempre que algo de "fato no negócio" deva ter ≥1 reação independente.
- **Outbox pattern** quando não pode perder evento (pagamento, fiscal). É boilerplate justificado.
- **CQRS** quando consultas pesadas degradam o modelo de escrita ou exigem joins/projeções pesadas. **Não** é "obrigatório" em DDD.
- **Não usar Event Sourcing** sem necessidade clara. ES é caro de operar; CQRS sem ES já resolve 90%.

## Armadilhas comuns

- Despachar evento **antes** de salvar — handler reage a algo que pode nunca ter persistido.
- Handler síncrono crítico que falha e silencia. Use outbox + worker.
- Read model "esquecido" e divergindo do estado real. Eventos atualizam read models — falha de event handling = read model defasado.
- Tratar Read Model como entidade — read model é DTO. Sem comportamento.

## Links relacionados

- [[Aggregates]]
- [[Padroes/Observer]]
- [[Performance/Filas]]

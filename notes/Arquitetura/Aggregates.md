---
tags: [php, arquitetura, ddd, aggregate]
fase: 4
status: draft
---

# Aggregates

## Conceito

O conceito de *Aggregate* aparece em **Eric Evans** (*DDD*, 2003, cap. 6, "*The Life Cycle of a Domain Object*") como o padrão para **resolver consistência em modelos complexos com muitas entidades relacionadas**. **Vaughn Vernon** refinou e popularizou regras práticas em sua série "*Effective Aggregate Design*" (2011) e em *Implementing DDD* (2013), capítulo 10.

Definição: um agregado é um **cluster de objetos (entidades + VOs) tratado como uma única unidade de consistência**. Tem uma **raiz** (*Aggregate Root*, AR) — a única entidade que o mundo externo referencia. Toda interação com o cluster passa pela raiz. Invariantes do cluster (regras que precisam ser sempre verdadeiras) são **garantidos pela raiz**, não dispersos.

```
                    ┌────────────────────────────────────────┐
                    │          Aggregate "Order"             │
                    │                                        │
                    │   ┌──────────────┐  ◀── Aggregate Root │
                    │   │    Order     │                     │
                    │   │              │                     │
                    │   │ + items      │  controla acesso ▼  │
                    │   │ + status     │                     │
                    │   │ + total()    │  ┌────────────┐     │
                    │   │ + addItem()  │──▶│ OrderItem  │    │
                    │   │ + cancel()   │  ├────────────┤     │
                    │   │              │──▶│ OrderItem  │    │
                    │   └──────────────┘  └────────────┘     │
                    │                                        │
                    │   referência a OUTROS agregados        │
                    │   só por ID:  CustomerId, ProductId    │
                    │                                        │
                    └────────────────────────────────────────┘
```

A regra de ouro é o **consistency boundary**: uma transação muta **um único agregado**. Se você precisa mutar `Order` e `Inventory` no mesmo commit, ou é o mesmo agregado, ou (mais provável) são agregados diferentes e a consistência entre eles é **eventual** — coordenada por eventos de domínio.

Vernon condensou em **quatro regras práticas** ("Effective Aggregate Design"):

1. **Modele invariantes verdadeiros dentro de fronteiras de consistência.** Só inclua no agregado o que precisa ser consistente atomicamente. Tudo o mais sai.
2. **Projete agregados pequenos.** Agregado idealmente cabe na memória sem esforço, carrega rápido, locka pouco. "*Customer* com todas as suas *Orders*" é quase sempre errado — orders são agregados separados.
3. **Referencie outros agregados por identidade.** `Order` carrega `CustomerId`, não `Customer`. Isso evita "carregar o mundo" e desacopla persistência.
4. **Use consistência eventual fora do agregado.** O que precisa ser eventualmente consistente vira *Domain Event* + handler em outro agregado.

A **Aggregate Root** é o único ponto de entrada. O repositório carrega e salva agregados **inteiros**, sempre pela raiz. Não existe `OrderItemRepository` — items vivem dentro do `Order` e são acessados via `Order`. Cada agregado tem **um repositório**.

Outro ponto crítico é **concorrência otimista**. Se dois usuários carregam o mesmo `Order` e mutam em paralelo, o segundo `save` deve falhar — ou perdemos uma das mutações. O padrão é versionar o agregado (`version: int`) e o `save` fazer `UPDATE ... WHERE id = ? AND version = ?`; se 0 linhas afetadas, conflito.

## Por que importa

Sem agregados como unidade de consistência, o que acontece em sistemas reais:

1. **Invariantes dispersos.** A regra "não pode adicionar item em pedido confirmado" aparece no controller, no service A, no service B. Algum dia alguém adiciona um caminho novo e esquece. Pedido confirmado ganha item novo. Cliente reclama. Bug "fantasma" que ninguém reproduz.

2. **Race conditions inevitáveis.** Dois requests simultâneos ajustam o mesmo pedido sem sincronização. Total fica divergente do somatório de items. Sem AR + versionamento, não há ponto onde detectar o conflito.

3. **Lock contention.** Agregado mal-projetado (gigante) gera locks longos. `Customer` com 5000 orders, todas carregadas, todas serializadas — uma operação trivial bloqueia outras.

4. **Persistência inconsistente.** Salvar pedaço de cluster (só a `Order`, esquecendo `OrderItem`) gera estado inválido em disco. Repositório por AR resolve: salva o cluster inteiro de uma vez.

5. **Domínio anêmico.** Sem AR concentrando comportamento, regras migram para services. Surge `OrderService::addItem(Order $o, OrderItem $i)` — não há diferença entre objeto e struct. Domínio vira CRUD com paint job.

Em 2-3 anos, sistema com agregados bem desenhados tem **muitos pontos de mutação validados**; sistema sem tem **muitos caminhos não validados** que coexistem e divergem.

## Exemplo de código PHP

### 1. Aggregate Root com invariantes protegidos

```php
<?php
declare(strict_types=1);

namespace App\Domain\Order;

use App\Domain\Shared\Money;
use App\Domain\Customer\CustomerId;
use App\Domain\Product\ProductId;
use App\Domain\Order\Events\{OrderPaid, OrderCancelled};

final class Order
{
    /** @var list<OrderItem> */
    private array $items = [];

    /** @var list<object> */
    private array $pendingEvents = [];

    private int $version = 0;   // optimistic lock

    public function __construct(
        public readonly OrderId $id,
        public readonly CustomerId $customer,
        private OrderStatus $status,
        public readonly \DateTimeImmutable $createdAt,
    ) {}

    public static function place(OrderId $id, CustomerId $customer, \DateTimeImmutable $now): self
    {
        return new self($id, $customer, OrderStatus::Pending, $now);
    }

    // Mundo externo NÃO toca items diretamente — passa pela raiz
    public function addItem(ProductId $product, int $qty, Money $unitPrice): void
    {
        // Invariante protegido aqui — único ponto de mutação
        if ($this->status !== OrderStatus::Pending) {
            throw new \DomainException('cannot modify non-pending order');
        }
        if ($qty <= 0) {
            throw new \InvalidArgumentException('qty must be positive');
        }
        if ($this->itemCount() >= 100) {
            throw new \DomainException('order item limit reached');
        }

        $this->items[] = new OrderItem($product, $qty, $unitPrice);
    }

    public function pay(string $providerTransactionId, \DateTimeImmutable $now): void
    {
        if (!$this->status->canTransitionTo(OrderStatus::Paid)) {
            throw new InvalidStateTransition($this->status, OrderStatus::Paid);
        }
        if ($this->items === []) {
            throw new \DomainException('cannot pay empty order');
        }

        $this->status = OrderStatus::Paid;
        $this->pendingEvents[] = new OrderPaid(
            $this->id,
            $this->total(),
            $providerTransactionId,
            $now,
        );
    }

    public function cancel(string $reason, \DateTimeImmutable $now): void
    {
        if (!$this->status->canTransitionTo(OrderStatus::Cancelled)) {
            throw new InvalidStateTransition($this->status, OrderStatus::Cancelled);
        }

        $this->status = OrderStatus::Cancelled;
        $this->pendingEvents[] = new OrderCancelled($this->id, $reason, $now);
    }

    public function total(): Money
    {
        return array_reduce(
            $this->items,
            fn (Money $acc, OrderItem $i) => $acc->add($i->subtotal()),
            new Money(0, 'BRL'),
        );
    }

    public function itemCount(): int
    {
        return count($this->items);
    }

    public function status(): OrderStatus
    {
        return $this->status;
    }

    public function version(): int
    {
        return $this->version;
    }

    // Devolve cópia — não permite mutação externa
    /** @return list<OrderItem> */
    public function items(): array
    {
        return $this->items;
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

### 2. Entity interna ao agregado (NÃO é AR)

```php
<?php
declare(strict_types=1);

namespace App\Domain\Order;

use App\Domain\Shared\Money;
use App\Domain\Product\ProductId;

// Entidade local: tem identidade dentro do agregado, mas NÃO é AR
// Não é acessada de fora — só via Order
final class OrderItem
{
    public function __construct(
        public readonly ProductId $product,
        public readonly int $qty,
        public readonly Money $unitPrice,
    ) {}

    public function subtotal(): Money
    {
        return $this->unitPrice->multiply($this->qty);
    }
}
```

### 3. Repository por agregado, com optimistic locking

```php
<?php
declare(strict_types=1);

namespace App\Domain\Order;

interface OrderRepository
{
    public function ofId(OrderId $id): ?Order;

    /**
     * @throws ConcurrencyException quando version no banco != version do AR
     */
    public function save(Order $order): void;
}
```

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Persistence;

use App\Domain\Order\{Order, OrderId, OrderRepository, ConcurrencyException};

final readonly class PdoOrderRepository implements OrderRepository
{
    public function __construct(private \PDO $pdo) {}

    public function ofId(OrderId $id): ?Order
    {
        // hidrata o AR inteiro: order + items numa única consulta (join)
        // ... omitido por brevidade ...
    }

    public function save(Order $order): void
    {
        $stmt = $this->pdo->prepare(
            'UPDATE orders SET status = :status, version = version + 1
             WHERE id = :id AND version = :expected_version'
        );
        $stmt->execute([
            'status'           => $order->status()->value,
            'id'               => $order->id->value,
            'expected_version' => $order->version(),
        ]);

        // Optimistic lock: 0 linhas = alguém mutou em paralelo
        if ($stmt->rowCount() === 0) {
            throw new ConcurrencyException($order->id);
        }

        // ... persiste items (delete + insert dentro da mesma transação)
    }
}
```

### 4. Referência entre agregados — por ID, nunca por objeto

```php
<?php
declare(strict_types=1);

// RUIM — agregado carrega outro agregado inteiro
final class Order
{
    public function __construct(
        public readonly OrderId $id,
        public readonly Customer $customer,   // <-- outro agregado, errado
    ) {}
}

// BOM — agregado referencia OUTRO agregado por ID
final class Order
{
    public function __construct(
        public readonly OrderId $id,
        public readonly CustomerId $customer, // <-- apenas o ID
    ) {}
}
```

Para enriquecer com dados do `Customer` na leitura, use *read model* / *query* — não carregue o agregado vizinho.

### 5. Consistência eventual entre agregados via evento

```php
<?php
declare(strict_types=1);

// Agregado Order emite OrderPaid; agregado Inventory reage em handler separado.
// NÃO é "Order.pay() chama Inventory.reserve()" — isso violaria a fronteira.

namespace App\Application\Inventory;

use App\Domain\Order\Events\OrderPaid;
use App\Domain\Inventory\InventoryRepository;

final readonly class ReserveInventoryOnOrderPaid
{
    public function __construct(private InventoryRepository $inventory) {}

    public function __invoke(OrderPaid $event): void
    {
        // Transação separada, agregado separado, eventualmente consistente
        $inventory = $this->inventory->ofId($event->productId);
        $inventory->reserve($event->qty);
        $this->inventory->save($inventory);
    }
}
```

## Quando usar / quando não usar

- **Use** sempre que há invariantes entre múltiplas entidades que precisam ser garantidos atomicamente (Order + Items + Coupons).
- **Use** quando há concorrência real sobre os mesmos dados (sistemas com múltiplos atores escrevendo).
- **Mantenha pequeno.** Agregado gigante (`Customer` com 5000 orders) = lock contention + memória. Quando em dúvida, separe.
- **Referencie por ID.** Nunca por objeto. Carrega só o que vai mutar.
- **Não use** em CRUD trivial sem invariantes. `User { id, name, email }` com endpoint para alterar nome — não precisa ser agregado, é registro.
- **Não force** agregados em consultas. Para listar dados, use *read model*; agregado é para **escrever**.

## Armadilhas comuns

- **Carregar agregado pela metade.** "Só quero o status, não os items". Quebra invariantes — se mutar e salvar, perde items. Repository **sempre** devolve o AR inteiro. Se está pesado, repense a fronteira do agregado.
- **Invariante validado fora do AR.** `if ($order->status === 'pending') $order->items[] = ...` no Use Case. Em algum momento outro caminho esquece o `if`. Volte a regra para o AR.
- **`getItems()` retornando referência mutável.** PHP arrays são por valor, mas se a entidade interna for objeto mutável e exposta, o cliente externo pode mutar. Devolva cópia ou tipo imutável.
- **Agregado gigante.** `Customer` com `addresses`, `orders`, `paymentMethods`, `subscriptions`, `tickets`. Carregar isso para mudar o nome é desperdício e lock contention. Cada um vira agregado próprio ou entidade local pequena.
- **Sem optimistic locking.** Sistema com concorrência sem versão no agregado — "lost update" silencioso. Sempre versione o AR e cheque no `save`.
- **Repository genérico.** `Repository::findBy(['status' => 'paid'])` quebra encapsulamento — qualquer chamador pode pedir qualquer combinação. Métodos com intenção: `ofId`, `pendingOlderThan`, `paidByCustomer`.
- **Múltiplos agregados na mesma transação.** Se você precisa abrir transação que muta `Order` E `Inventory`, **um deles está modelado errado** ou deveria ser consistência eventual via evento.
- **Eventos despachados antes de salvar.** Handler reage a algo que pode falhar no commit. Sempre: salvar primeiro (com eventos na mesma transação via outbox), despachar depois.
- **AR exposto via JSON serialization automática.** `json_encode($order)` despeja estrutura interna. Use DTOs/Read Models para output.

## Links relacionados

- [[Value-Objects]]
- [[Use-Cases]]
- [[Domain-Events-CQRS]]
- [[Camadas-DDD]]
- [[Hexagonal]]
- [[Padroes/Repository]]

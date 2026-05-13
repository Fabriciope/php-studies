---
tags: [php, padroes, persistencia]
fase: 3
status: draft
---

# Repository

## Conceito

Repository é um padrão tático do **DDD** (Eric Evans, 2003), também referenciado por Fowler em *PoEAA*. Define uma **abstração tipo "coleção em memória" sobre a persistência de agregados**. O domínio fala `$orders->save($order)` e `$orders->ofId($id)`; quem implementa pode ser MySQL, in-memory, arquivo, API externa. O domínio não sabe nem se importa.

A interface do repository expressa **conceitos do domínio**, não detalhes de armazenamento. Métodos chamam-se `pendingOlderThan(...)`, `topSellersInPeriod(...)`, `ofCustomer(...)` — não `findBy(['status' => 'pending'])`. A linguagem ubíqua atravessa a fronteira.

```
+----------------------+         +----------------------+
|  UseCase / Domínio   | ----->  |  OrderRepository     |
|  (não sabe de SQL)   |         |  (interface)         |
+----------------------+         +----------+-----------+
                                            ^
                              +-------------+-------------+
                              |                           |
                     PdoOrderRepository          InMemoryOrderRepository
                     (produção)                  (testes)
```

Distinções importantes:

| Padrão | Posição | Característica |
|---|---|---|
| **Repository** | Domínio fala com ele via interface | Coleção de agregados; vocabulário do domínio |
| **DAO** (Data Access Object) | Camada de persistência | Métodos refletem operações SQL (`findById`, `findByX`, `update`); CRUD orientado a tabelas |
| **Data Mapper** | Tradutor entidade ↔ linha | Entidade não sabe persistir; mapper faz a ponte (Doctrine ORM) |
| **Active Record** | Entidade carrega persistência | `$user->save()`; mistura modelo e storage (Eloquent) |

Repository **usa** Data Mapper internamente quando há ORM por trás. DAO é mais primitivo e amarrado ao schema. Active Record é o oposto filosófico do Repository — Eloquent's `Model` faz tudo, o que é produtivo em CRUD e doloroso em domínio rico.

**Generic Repository** (com `<T>` simulado por docblock) vs **Specific Repository**:

- Generic: `interface Repository<T> { save(T); ofId(Id): ?T; }` — útil para CRUD trivial, mas anêmico para domínio rico.
- Specific: cada agregado tem seu repository nomeado com métodos próprios. Preferível em DDD.

**Composição com [[Specification]]**: quando a explosão de `findBy*` ameaça (`findActiveByRegion`, `findActivePremium`, `findInactiveOlderThan`), aceitar uma `Specification` resolve sem multiplicar métodos:

```php
$orders->matching(new IsActive()->and(new InRegion('SP')));
```

**Read Model vs Write Repository (CQRS leve)**: para leituras complexas (listagens, dashboards), o Repository do agregado é mau encaixe — ele carrega o agregado inteiro. Use **Read Model** (DTO + query direta) separado. O Repository fica focado em ciclo de vida do agregado (load + save), o Read Model em projeções para UI.

**Identity Map**: garantir que `$repo->ofId('123')` chamado duas vezes na mesma unidade de trabalho devolva a **mesma instância**. Doctrine implementa isso internamente. Em repositórios manuais, considere implementar se há risco de duas versões coexistirem no mesmo request.

## Por que importa

Sem Repository, regras de negócio dependem direto de SQL ou ORM. Os sintomas:

- **Testes lentos**: cada teste sobe banco, transação, rollback.
- **Troca de storage = projeto**: trocar MySQL por PostgreSQL, ou parte por DynamoDB, exige rever centenas de queries no domínio.
- **Vazamento de detalhe**: `$query->whereNull('deleted_at')->where(...)` em use cases obriga o autor do use case a entender o schema.
- **Acoplamento ao ORM**: trocar Doctrine por Eloquent (ou vice-versa) vira reescrita.

Cenários concretos:

- **API de pedidos**: o use case `CheckoutOrder` recebe `OrderRepository`. Em produção, Pdo. Em testes unitários, InMemory. Em integração, real. Sem mudar o use case.
- **Migração gradual**: durante migração de Doctrine para algo mais leve, o repository abstrai e o switch é transparente.
- **Multi-tenant**: o repository sabe filtrar por tenant; o use case não.

## Exemplo de código PHP

### 1) Interface segregada por papel

```php
<?php
declare(strict_types=1);

// ISP aplicada: leitores e escritores em interfaces separadas.
interface OrderReader
{
    public function ofId(OrderId $id): ?Order;

    /** @return iterable<Order> */
    public function pendingOlderThan(\DateTimeImmutable $cutoff): iterable;
}

interface OrderWriter
{
    public function save(Order $order): void;
    public function delete(OrderId $id): void;
}

// Implementação concreta pode realizar as duas — clientes só dependem do que usam.
interface OrderRepository extends OrderReader, OrderWriter {}
```

### 2) Adapter PDO (produção)

```php
<?php
declare(strict_types=1);

final readonly class PdoOrderRepository implements OrderRepository
{
    public function __construct(
        private \PDO $pdo,
        private OrderHydrator $hydrator,
    ) {}

    public function save(Order $order): void
    {
        $stmt = $this->pdo->prepare(
            'INSERT INTO orders (id, status, total_cents, currency, created_at)
             VALUES (?,?,?,?,?)
             ON DUPLICATE KEY UPDATE
                status     = VALUES(status),
                total_cents = VALUES(total_cents)',
        );
        $stmt->execute([
            $order->id->toString(),
            $order->status->value,
            $order->total->cents,
            $order->total->currency,
            $order->createdAt->format('Y-m-d H:i:s'),
        ]);
    }

    public function ofId(OrderId $id): ?Order
    {
        $stmt = $this->pdo->prepare('SELECT * FROM orders WHERE id = ?');
        $stmt->execute([$id->toString()]);
        $row = $stmt->fetch(\PDO::FETCH_ASSOC);
        return $row ? $this->hydrator->fromRow($row) : null;
    }

    public function pendingOlderThan(\DateTimeImmutable $cutoff): iterable
    {
        $stmt = $this->pdo->prepare(
            'SELECT * FROM orders WHERE status = ? AND created_at < ?',
        );
        $stmt->execute(['pending', $cutoff->format('Y-m-d H:i:s')]);
        while ($row = $stmt->fetch(\PDO::FETCH_ASSOC)) {
            yield $this->hydrator->fromRow($row);
        }
    }

    public function delete(OrderId $id): void
    {
        $this->pdo->prepare('DELETE FROM orders WHERE id = ?')
            ->execute([$id->toString()]);
    }
}
```

### 3) Adapter In-Memory (testes)

```php
<?php
declare(strict_types=1);

final class InMemoryOrderRepository implements OrderRepository
{
    /** @var array<string, Order> */
    private array $store = [];

    public function save(Order $order): void
    {
        $this->store[$order->id->toString()] = $order;
    }

    public function ofId(OrderId $id): ?Order
    {
        return $this->store[$id->toString()] ?? null;
    }

    public function pendingOlderThan(\DateTimeImmutable $cutoff): iterable
    {
        foreach ($this->store as $order) {
            if ($order->status === OrderStatus::Pending
                && $order->createdAt < $cutoff
            ) {
                yield $order;
            }
        }
    }

    public function delete(OrderId $id): void
    {
        unset($this->store[$id->toString()]);
    }
}
```

### 4) Composição com Specification (matching)

```php
<?php
declare(strict_types=1);

// Em vez de proliferar findBy*, aceite uma Specification.
interface OrderRepository extends OrderReader, OrderWriter
{
    /** @return iterable<Order> */
    public function matching(Specification $spec): iterable;
}

// Spec específica do domínio
final readonly class OrderIsPending implements Specification
{
    public function isSatisfiedBy(mixed $candidate): bool
    {
        return $candidate instanceof Order
            && $candidate->status === OrderStatus::Pending;
    }
}

// Uso composto
$old = $orders->matching(
    (new OrderIsPending())
        ->and(new CreatedBefore($cutoff)),
);
```

### 5) Read Model separado (CQRS leve)

```php
<?php
declare(strict_types=1);

// Para listagem em dashboard, não use o Repository do agregado.
// Crie um Read Model dedicado com DTO otimizado.

final readonly class OrderListItem
{
    public function __construct(
        public string $id,
        public string $customerName,
        public int $totalCents,
        public string $status,
        public string $createdAt,
    ) {}
}

interface OrderListReader
{
    /** @return iterable<OrderListItem> */
    public function paginate(int $page, int $perPage, ?string $status = null): iterable;
}

final readonly class PdoOrderListReader implements OrderListReader
{
    public function __construct(private \PDO $pdo) {}

    public function paginate(int $page, int $perPage, ?string $status = null): iterable
    {
        // SQL otimizado com JOIN, sem carregar agregado inteiro.
        $sql = 'SELECT o.id, c.name, o.total_cents, o.status, o.created_at
                FROM orders o JOIN customers c ON c.id = o.customer_id';
        // ...
    }
}
```

## Quando usar / quando não usar

- **Usar sempre** entre domínio e persistência em arquitetura em camadas, hexagonal ou DDD tático.
- **Usar** quando o sistema tem regras de negócio reais sobre os agregados — não apenas CRUD.
- **Usar** quando você quer testes unitários do use case sem subir banco.
- **Composição com [[Specification]]**: aplicar quando os "find*" começam a multiplicar; melhor uma `matching(Spec)` que oito `findBy*`.
- **Não usar** em scripts CRUD simples sem regra — ali o ORM **é** o modelo, e Repository vira cerimônia vazia (Laravel/Eloquent em CRUD: o Model basta).
- **Não usar** para listagens complexas — use Read Model separado.
- **Não virar DAO**: se cada query do sistema gera um método no repository, perdeu-se o ponto. Repository expressa **conceitos de domínio**.

## Armadilhas comuns

- **`findBy(['status' => 'paid'])` na interface**: aceitar mapa genérico vaza schema. Prefira métodos nomeados de domínio. Para múltiplos critérios, `matching(Specification)`.
- **Expor Query Builder do ORM**: `OrderRepository::query(): QueryBuilder` quebra a abstração. O caller passa a depender do ORM.
- **Vazar Doctrine `Collection` no retorno**: tipos do ORM no contrato do domínio. Retorne `iterable` ou `array` de entidades.
- **Repository carregando agregado parcial**: `loadWithoutItems()` para "otimizar" — agregado tem que estar consistente, ou é um Read Model disfarçado.
- **`save()` que cascateia errado**: salvar um pedido salva também produtos catalogados? Não — agregado tem fronteira; salve só o que pertence.
- **Esquecer transação no caller**: o repository não abre transação por si. Use Unit of Work ou abra explicitamente no use case quando há múltiplos saves.
- **Identity Map ausente em domínio rico**: dois carregamentos do mesmo `$id` devolvem duas instâncias divergentes; modificações em uma não aparecem na outra. Em Doctrine, o ORM cuida; em código manual, considere.
- **Generic Repository em todo agregado**: vira anêmico. Específico expressa o domínio melhor.

## Links relacionados

- [[Specification]]
- [[Decorator]]
- [[Factory]]
- [[Strategy]]
- [[Padroes/SOLID]]
- [[Arquitetura/Hexagonal]]
- [[Arquitetura/Aggregates]]
- [[Arquitetura/Domain-Events-CQRS]]

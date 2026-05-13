---
tags: [php, padroes, specification, pipeline]
fase: 3
status: draft
---

# Specification (+ Pipeline + DI)

## Conceito

Esta nota agrupa três padrões que aparecem juntos na arquitetura de aplicações orientadas a casos de uso: **Specification** (predicado composto), **Pipeline** (esteira de transformações) e **DI / PSR-11** (container de inversão de controle). Os três são ferramentas distintas, mas se reforçam mutuamente.

### Specification

Originalmente descrito por Eric Evans e Martin Fowler como padrão tático de DDD. **Encapsula uma regra de seleção ou validação como objeto**: `$spec->isSatisfiedBy($candidate): bool`. O ganho real vem da **composição** com operadores lógicos:

```
spec.and(other)   -> AndSpec  (ambos satisfeitos)
spec.or(other)    -> OrSpec   (ao menos um)
spec.not()        -> NotSpec  (negação)
```

Specifications podem ter três usos:

1. **Validação**: "este pedido pode ser despachado?" — `CanShipSpec::isSatisfiedBy($order)`.
2. **Seleção** (filtro de coleção): `array_filter($orders, fn ($o) => $spec->isSatisfiedBy($o))`.
3. **Tradução para consulta**: spec sabe gerar SQL/Criteria — repository delega; veja **Specification em consultas** abaixo.

### Pipeline

Encadeia transformações: cada **stage** recebe um contexto e devolve um contexto (igual ou modificado), formando uma esteira. É o irmão funcional de Decorator/Middleware, com foco em fluxo de dados, não em interceptação:

```
ctx ──> [stage 1] ──> ctx' ──> [stage 2] ──> ctx'' ──> [stage 3] ──> resultado
```

Comparado a Decorator: Decorator preserva interface e empilha por composição transparente; Pipeline tem semântica explícita de "etapa de processamento" e ordem definida. Em CommandBus/QueryBus modernos é comum chamar de Pipeline embora estrutura interna seja Decorator.

### DI / PSR-11

A PSR-11 (`Psr\Container\ContainerInterface`) padroniza o container de DI: `get($id)` e `has($id)`. Container é detalhe de **wiring**; código de domínio nunca o consulta. A diferença crítica:

- **Constructor Injection**: dependências chegam pelo construtor; classe declara o que precisa.
- **Service Locator (anti-pattern)**: classe recebe o container e pede `$container->get(...)` — esconde dependências, dificulta teste, viola DIP.

## Por que importa

- **Specification** mata `if` aninhado de regras de negócio e permite reuso direto: `elegibleForFreeShipping->and($brazilOnly)->isSatisfiedBy($order)`. A mesma regra serve filtragem, validação, autorização.
- **Pipeline** dá forma a fluxos com etapas claras e ordenadas (validar → autorizar → calcular impostos → aplicar desconto → fechar). Cada stage é independente, testável, opcionalmente plugável.
- **PSR-11** torna o container intercambiável (Symfony, PHP-DI, Pimple, Laminas, Laravel) e o teste possível sem amarrar à infra de DI.

Cenários concretos:

- **Frete grátis**: regra "pedido > R$ 200 e cliente Premium e CEP em região atendida" — três Specs compostas com `and`. Reuso em filtro de catálogo, em validação no checkout e em badge de UI.
- **Use case CheckoutOrder**: Pipeline com stages `ValidateCart`, `ComputeTax`, `ApplyDiscount`, `ChargePayment`, `PersistOrder`. Cada um isolado.
- **Multi-tenant**: PSR-11 com `ContainerFactory::forTenant($id)` devolvendo container configurado por tenant.

## Exemplo de código PHP

### 1) Specification base com composição

```php
<?php
declare(strict_types=1);

interface Specification
{
    public function isSatisfiedBy(mixed $candidate): bool;
    public function and(Specification $other): Specification;
    public function or(Specification $other): Specification;
    public function not(): Specification;
}

abstract class CompositeSpec implements Specification
{
    final public function and(Specification $other): Specification
    {
        return new AndSpec($this, $other);
    }

    final public function or(Specification $other): Specification
    {
        return new OrSpec($this, $other);
    }

    final public function not(): Specification
    {
        return new NotSpec($this);
    }
}

final class AndSpec extends CompositeSpec
{
    public function __construct(
        private readonly Specification $a,
        private readonly Specification $b,
    ) {}

    public function isSatisfiedBy(mixed $c): bool
    {
        return $this->a->isSatisfiedBy($c) && $this->b->isSatisfiedBy($c);
    }
}

final class OrSpec extends CompositeSpec
{
    public function __construct(
        private readonly Specification $a,
        private readonly Specification $b,
    ) {}

    public function isSatisfiedBy(mixed $c): bool
    {
        return $this->a->isSatisfiedBy($c) || $this->b->isSatisfiedBy($c);
    }
}

final class NotSpec extends CompositeSpec
{
    public function __construct(private readonly Specification $inner) {}

    public function isSatisfiedBy(mixed $c): bool
    {
        return !$this->inner->isSatisfiedBy($c);
    }
}
```

### 2) Specifications concretas

```php
<?php
declare(strict_types=1);

final class TotalAbove extends CompositeSpec
{
    public function __construct(private readonly Money $threshold) {}

    public function isSatisfiedBy(mixed $cart): bool
    {
        return $cart instanceof Cart
            && $cart->subtotal()->cents >= $this->threshold->cents;
    }
}

final class CustomerIsPremium extends CompositeSpec
{
    public function isSatisfiedBy(mixed $customer): bool
    {
        return $customer instanceof Customer
            && $customer->tier === CustomerTier::Premium;
    }
}

final class ShipsToBrazil extends CompositeSpec
{
    public function isSatisfiedBy(mixed $address): bool
    {
        return $address instanceof Address
            && $address->countryCode === 'BR';
    }
}

// Composição expressiva:
$freeShipping = (new TotalAbove(Money::brl(200)))
    ->and(new CustomerIsPremium())
    ->and(new ShipsToBrazil());

if ($freeShipping->isSatisfiedBy($context)) {
    // ...
}
```

### 3) Specification traduzível para consulta

```php
<?php
declare(strict_types=1);

// Specification que sabe se traduzir para SQL. Útil para passar ao Repository
// sem proliferar findBy* nem carregar tudo na memória.

interface SqlSpecification extends Specification
{
    /** @return array{0: string, 1: array<string, mixed>} where + params */
    public function toSqlWhere(): array;
}

final class OrderIsPending implements SqlSpecification
{
    public function isSatisfiedBy(mixed $c): bool
    {
        return $c instanceof Order && $c->status === OrderStatus::Pending;
    }

    public function toSqlWhere(): array
    {
        return ['status = :status', ['status' => 'pending']];
    }

    public function and(Specification $o): Specification { /* ... */ throw new \LogicException(); }
    public function or(Specification $o): Specification { /* ... */ throw new \LogicException(); }
    public function not(): Specification { /* ... */ throw new \LogicException(); }
}

// Repository usa a tradução:
final readonly class PdoOrderRepository implements OrderRepository
{
    public function __construct(private \PDO $pdo) {}

    /** @return iterable<Order> */
    public function matching(SqlSpecification $spec): iterable
    {
        [$where, $params] = $spec->toSqlWhere();
        $stmt = $this->pdo->prepare("SELECT * FROM orders WHERE {$where}");
        $stmt->execute($params);
        while ($row = $stmt->fetch(\PDO::FETCH_ASSOC)) {
            yield Order::fromRow($row);
        }
    }
}
```

### 4) Pipeline com contexto imutável

```php
<?php
declare(strict_types=1);

final readonly class CheckoutContext
{
    public function __construct(
        public Cart $cart,
        public Customer $customer,
        public ?Money $tax = null,
        public ?Money $discount = null,
        public ?Money $finalTotal = null,
        public ?PaymentReceipt $receipt = null,
    ) {}

    public function withTax(Money $tax): self
    {
        return new self($this->cart, $this->customer, $tax, $this->discount, $this->finalTotal, $this->receipt);
    }

    public function withDiscount(Money $d): self
    {
        return new self($this->cart, $this->customer, $this->tax, $d, $this->finalTotal, $this->receipt);
    }

    // ... outros withers
}

interface Stage
{
    public function __invoke(CheckoutContext $ctx): CheckoutContext;
}

final class Pipeline
{
    /** @param list<Stage> $stages */
    public function __construct(private readonly array $stages) {}

    public function run(CheckoutContext $ctx): CheckoutContext
    {
        return array_reduce(
            $this->stages,
            static fn (CheckoutContext $c, Stage $s): CheckoutContext => $s($c),
            $ctx,
        );
    }
}
```

### 5) Stages concretos

```php
<?php
declare(strict_types=1);

final readonly class ComputeTaxStage implements Stage
{
    public function __construct(private TaxCalculator $calc) {}

    public function __invoke(CheckoutContext $ctx): CheckoutContext
    {
        $tax = $this->calc->compute($ctx.cart, $ctx->customer);
        return $ctx->withTax($tax);
    }
}

final readonly class ApplyDiscountStage implements Stage
{
    public function __construct(private DiscountStrategy $discount) {}

    public function __invoke(CheckoutContext $ctx): CheckoutContext
    {
        $discounted = $this->discount->apply($ctx->cart->subtotal());
        return $ctx->withDiscount($discounted);
    }
}

// Composição:
$pipeline = new Pipeline([
    new ValidateCartStage(),
    new ComputeTaxStage($taxCalc),
    new ApplyDiscountStage($discount),
    new ChargePaymentStage($gateway),
    new PersistOrderStage($repo),
]);

$result = $pipeline->run(new CheckoutContext($cart, $customer));
```

### 6) PSR-11 e injeção correta vs anti-pattern

```php
<?php
declare(strict_types=1);

use Psr\Container\ContainerInterface;

// CORRETO: dependências resolvidas pelo container, mas a classe declara
// o que precisa explicitamente no construtor.
final readonly class CheckoutController
{
    public function __construct(
        private CheckoutPipeline $pipeline,
        private DiscountFactory $discounts,
    ) {}

    public function __invoke(CheckoutRequest $req): CheckoutResponse { /* ... */ }
}

// Container faz wiring uma vez, no entrypoint:
$controller = $container->get(CheckoutController::class);

// ANTI-PATTERN: Service Locator.
// A classe esconde dependências; pega no container em runtime.
final class CheckoutControllerBad
{
    public function __construct(private ContainerInterface $container) {}

    public function handle(CheckoutRequest $req): CheckoutResponse
    {
        $pipeline = $this->container->get('checkout.pipeline');  // ruim
        $discount = $this->container->get('discount.factory');   // ruim
        // ...
    }
}
```

## Quando usar / quando não usar

- **Specification** quando a mesma regra aparece em 3+ lugares (filtro, validação, autorização) ou quando regras compõem.
- **Specification com tradução SQL** quando o repository teria que multiplicar `findBy*` e a regra precisa rodar no banco por volume.
- **Pipeline** quando o fluxo tem ≥3 etapas claras e ordenadas, com estado que se acumula.
- **Pipeline com contexto imutável** sempre que possível — facilita raciocínio e testes.
- **DI Container (PSR-11)** sempre em produção; em testes unitários, instanciar manualmente é mais legível e rápido.
- **Não usar Specification** para regra com **uma** condição simples — `if` é mais honesto.
- **Não usar Pipeline** para fluxos de 2 passos — duas chamadas explícitas são mais claras.
- **Não usar container** dentro do código de domínio — service locator é anti-pattern.

## Armadilhas comuns

- **Specification que faz I/O**: `isSatisfiedBy` consultando banco — quebra reuso, vira regra opaca. Se precisa, é outra coisa (Policy + Repository, ou Specification que **devolve consulta** sem executar).
- **Specification que retorna mais que bool**: `isSatisfiedBy` devolvendo motivo da rejeição mistura predicado com diagnóstico. Para "por que falhou", use Result type ou ValidationResult separada.
- **Composição infinita sem nome**: `$a->and($b)->or($c->not())->and($d)` sem extrair em variável significativa. Dê nomes intermediários (`$elegibleForFreeShipping = ...`).
- **Pipeline mutável**: stages alterando estado compartilhado por referência. Bug sutil em concorrência e em rerun. Use contexto imutável (`readonly`) e retorne nova versão.
- **Pipeline cuja ordem importa mas não está documentada**: stages comutativos sem aviso, ou ordem implícita por configuração. Documente a ordem e a razão.
- **Container como Service Locator**: classe pedindo `$container->get(...)` direto. Dependências escondidas. Sempre injete dependências resolvidas no construtor.
- **Container em testes**: subir container completo só para instanciar uma classe é desperdício. Use `new` direto com stubs/mocks.
- **PSR-11 `has()` para fluxo de negócio**: `has($id)` é para detecção de presença em wiring, não para `if (tem) faça X else Y` de domínio. Domínio não fala com container.

## Links relacionados

- [[Strategy]]
- [[Repository]]
- [[Factory]]
- [[Decorator]]
- [[Observer]]
- [[Padroes/SOLID]]
- [[Arquitetura/Use-Cases]]
- [[Arquitetura/Hexagonal]]

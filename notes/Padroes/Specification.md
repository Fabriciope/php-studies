---
tags: [php, padroes, specification, pipeline]
fase: 3
status: stub
---

# Specification (+ Pipeline + DI)

## Conceito

**Specification** encapsula uma regra de seleção/validação como objeto: `$spec->isSatisfiedBy($candidate): bool`. Specs combinam (`and`/`or`/`not`).

**Pipeline** encadeia transformações: cada stage recebe e devolve, formando uma esteira (igual middlewares HTTP, mas para domínio).

**DI (PSR-11)** padroniza container: `ContainerInterface::get($id)` e `has($id)`. Container é detalhe de wiring; código de domínio nunca o consulta.

## Por que importa

- Specification: mata `if` aninhado de regras de negócio e permite reuso (`elegibleForFreeShipping->and($brazilOnly)`).
- Pipeline: dá forma a fluxos com etapas claras (validar → calcular impostos → aplicar desconto → fechar).
- PSR-11: torna o container intercambiável e o teste possível sem tocar serviço de DI.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// Specification
interface Specification {
    public function isSatisfiedBy(mixed $candidate): bool;
    public function and(self $other): self;
}

abstract class AbstractSpec implements Specification {
    final public function and(Specification $other): Specification {
        return new AndSpec($this, $other);
    }
}
final class AndSpec extends AbstractSpec {
    public function __construct(private Specification $a, private Specification $b) {}
    public function isSatisfiedBy(mixed $c): bool {
        return $this->a->isSatisfiedBy($c) && $this->b->isSatisfiedBy($c);
    }
}

final class TotalAbove extends AbstractSpec {
    public function __construct(private Money $threshold) {}
    public function isSatisfiedBy(mixed $cart): bool {
        return $cart instanceof Cart && $cart->subtotal()->cents >= $this->threshold->cents;
    }
}

// Pipeline
interface Stage {
    public function __invoke(CheckoutContext $ctx): CheckoutContext;
}

final class Pipeline {
    /** @param list<Stage> $stages */
    public function __construct(private array $stages) {}
    public function run(CheckoutContext $ctx): CheckoutContext {
        return array_reduce($this->stages, fn ($c, $s) => $s($c), $ctx);
    }
}

// DI / PSR-11
$container->get(CheckoutOrder::class)->__invoke($cmd);
```

## Quando usar / quando não usar

- **Specification** quando a mesma regra aparece em 3+ lugares (filtro, validação, autorização).
- **Pipeline** quando o fluxo tem ≥3 etapas claras e ordenadas.
- **Container** sempre em produção; em testes, instanciar manualmente é mais legível e rápido.
- **Não usar Specification** para regra com **uma** condição simples — `if` é mais honesto.

## Armadilhas comuns

- Specification que faz I/O (`isSatisfiedBy` consulta banco) — quebra reuso. Se precisa, é outra coisa (Policy + Repository).
- Container Service Locator: classe pedindo `$container->get(...)` direto — anti-pattern. Sempre injete dependências resolvidas no construtor.
- Pipeline mutável: stages alterando estado compartilhado por referência. Use contexto imutável (readonly) e devolva nova versão.

## Links relacionados

- [[Strategy]]
- [[Repository]]
- [[Arquitetura/Use-Cases]]

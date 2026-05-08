---
tags: [php, padroes, factory]
fase: 3
status: stub
---

# Factory

## Conceito

Centralizar a decisão de **qual** implementação criar e **como** construí-la. Pode ser método estático nomeado (`Order::fromCart()`), classe dedicada ou função pura. O caller pede o conceito, recebe a instância pronta.

## Por que importa

Construtor é "como nascer da própria essência". Quando um objeto pode nascer de múltiplas fontes (cart, dump, evento, JSON externo), espalhar essa lógica nos callers vaza detalhe. Factory concentra.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// 1) named constructors (factory methods estáticos)
final readonly class Money {
    public function __construct(public int $cents, public string $currency) {}

    public static function brl(float $reais): self {
        return new self((int) round($reais * 100), 'BRL');
    }
    public static function fromString(string $iso): self {
        // "BRL 19.90" -> Money
        [$cur, $val] = explode(' ', $iso);
        return new self((int) round(((float) $val) * 100), $cur);
    }
}

// 2) factory classe — escolhe estratégia a partir de input
final readonly class DiscountFactory {
    public function from(string $code, ?int $arg = null): DiscountStrategy {
        return match ($code) {
            'percent' => new PercentOff($arg ?? throw new InvalidArgumentException('arg')),
            'fixed'   => new FixedOff(Money::brl((float) $arg)),
            'none'    => new NoDiscount(),
            default   => throw new InvalidArgumentException("unknown discount: {$code}"),
        };
    }
}
```

## Quando usar / quando não usar

- **Named constructors** sempre que houver múltiplas formas de nascer um VO/entidade.
- **Factory class** quando a escolha depende de runtime/config e há ≥3 alternativas.
- **Não usar** quando o construtor já é claro e direto — `new Foo($a, $b)` em um único lugar não pede factory.
- **Não confundir com DI container** — container resolve grafo; factory cria por contexto.

## Armadilhas comuns

- Factory virar "god class" — cresce até saber de tudo. Quebre por aggregate/contexto.
- Construtor `private` + factory estática quebra hidratação por reflection/ORM antigos. Em geral `public`/`protected` resolve.

## Links relacionados

- [[Strategy]]
- [[Padroes/SOLID]]
- [[Arquitetura/Aggregates]]

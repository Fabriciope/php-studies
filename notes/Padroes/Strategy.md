---
tags: [php, padroes, strategy]
fase: 3
status: stub
---

# Strategy

## Conceito

Encapsular um algoritmo em uma interface; consumir através dela. Trocar de algoritmo = trocar de implementação. Substitui `if/elseif/switch` que selecionam comportamento por tipo de dado.

## Por que importa

É o padrão mais útil do dia-a-dia. Toda vez que `match($type)` retorna comportamentos diferentes, há um Strategy implícito esperando para nascer. Extraí-lo elimina a "escada de else" e libera teste isolado por estratégia.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

interface DiscountStrategy {
    public function apply(Money $total): Money;
}

final readonly class PercentOff implements DiscountStrategy {
    public function __construct(private int $percent) {}
    public function apply(Money $total): Money {
        $newCents = (int) ($total->cents * (100 - $this->percent) / 100);
        return new Money($newCents, $total->currency);
    }
}

final readonly class FixedOff implements DiscountStrategy {
    public function __construct(private Money $amount) {}
    public function apply(Money $total): Money {
        return new Money(max(0, $total->cents - $this->amount->cents), $total->currency);
    }
}

final readonly class NoDiscount implements DiscountStrategy {
    public function apply(Money $total): Money { return $total; }
}

// caller: agnóstico
function checkout(Cart $cart, DiscountStrategy $discount): Money {
    return $discount->apply($cart->subtotal());
}
```

## Quando usar / quando não usar

- **Toda vez que `match` ou `switch` selecionar lógica por tipo** — promova a Strategy.
- **Combina com [[Factory]]** que escolhe a estratégia a partir de configuração/input.
- **Não usar** se há só uma "estratégia" e a interface existe "para o futuro" — adicione quando a segunda nasce.
- **Não substitui Enum** quando o que varia é só **dado**. Strategy é para **comportamento**.

## Armadilhas comuns

- Estratégia carregando estado (cache, contadores) — viola substituibilidade. Mantenha pura.
- Confundir com [[Specification]]: Specification responde "este item satisfaz X?"; Strategy executa "como X deve ser feito?".

## Links relacionados

- [[Factory]]
- [[Specification]]
- [[Padroes/SOLID]]

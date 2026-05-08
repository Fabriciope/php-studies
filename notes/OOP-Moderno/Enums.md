---
tags: [php, oop, enums]
fase: 2
status: stub
---

# Enums

## Conceito

Enum em PHP 8.1+ é um tipo fechado. Pode ser **puro** (só cases) ou **backed** (cada case tem um valor escalar `int`/`string`). Suporta métodos, implementa interfaces, mas não pode ser instanciado fora do bloco `enum`.

## Por que importa

Substitui constantes "stringly typed" e classes de constantes. Em vez de `OrderStatus::PENDING = 'pending'` espalhado e qualquer string passar, o tipo `OrderStatus` impede valores inválidos no boundary.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

enum OrderStatus: string {
    case Pending = 'pending';
    case Paid    = 'paid';
    case Shipped = 'shipped';
    case Cancelled = 'cancelled';

    public function isFinal(): bool {
        return match ($this) {
            self::Shipped, self::Cancelled => true,
            default => false,
        };
    }

    public function label(): string {
        return match ($this) {
            self::Pending   => 'Aguardando pagamento',
            self::Paid      => 'Pago',
            self::Shipped   => 'Enviado',
            self::Cancelled => 'Cancelado',
        };
    }
}

// boundary: validação automática
$status = OrderStatus::from($input);   // throws ValueError se inválido
$maybe  = OrderStatus::tryFrom($input); // retorna null se inválido
```

## Quando usar / quando não usar

- **Sempre** que houver um conjunto fechado e conhecido em compile-time (status, tipos, papéis, dias).
- **Backed** quando precisa serializar/persistir; **pura** quando o valor não importa.
- **Não usar** quando o conjunto é dinâmico (carregado de banco/config) — aí continua sendo Value Object com `string` validada.

## Armadilhas comuns

- `match` exaustivo: se você não cobre todos os cases e usa `default` como fallback, perde o erro de compilação ao adicionar um case novo. Para forçar revisão futura, **omita** `default`.
- Enums **não** suportam construtor com lógica complexa — métodos são `match` puro.
- `from()` lança `ValueError`, não `TypeError`. Trate adequadamente.

## Links relacionados

- [[Sistema-de-Tipos]]
- [[Readonly-Classes]]
- [[Arquitetura/Aggregates]]

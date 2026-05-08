---
tags: [php, arquitetura, ddd, value-object]
fase: 4
status: stub
---

# Value Objects

## Conceito

Objeto imutável definido **por seus atributos**, não por identidade. Dois `Money(1990, 'BRL')` são **iguais** — não há "este Money" vs "aquele". Encapsula validação no construtor; toda mutação retorna nova instância.

## Por que importa

VOs eliminam a "primitive obsession": parâmetros `string $email`, `int $cents` que aceitam qualquer coisa. Um VO `Email` validado **uma vez** garante que toda função que recebe `Email` recebe um e-mail válido. A validação fica no tipo.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

final readonly class Email {
    public function __construct(public string $address) {
        if (!filter_var($address, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException("invalid email: {$address}");
        }
    }

    public function domain(): string {
        return substr($this->address, (int) strpos($this->address, '@') + 1);
    }

    public function equals(self $other): bool {
        return strtolower($this->address) === strtolower($other->address);
    }
}

final readonly class DateRange {
    public function __construct(
        public \DateTimeImmutable $from,
        public \DateTimeImmutable $to,
    ) {
        if ($to < $from) {
            throw new InvalidArgumentException('to before from');
        }
    }

    public function contains(\DateTimeImmutable $when): bool {
        return $when >= $this->from && $when <= $this->to;
    }

    public function days(): int {
        return (int) $this->from->diff($this->to)->days;
    }
}
```

## Quando usar / quando não usar

- **Sempre** que um conjunto de primitivas representar **um conceito** — Money, Email, CPF, DateRange, GeoPoint, OrderId.
- **Sempre** em domínio rico — se há regra associada a um valor, ele é VO.
- **Não usar** quando o valor é descartável e sem regra (loop counter).
- **Não usar** quando é uma string realmente livre (corpo de comentário).

## Armadilhas comuns

- VO mutável — perdeu o sentido. `readonly` por construção.
- `equals` ausente — comparar por `==` em PHP funciona em alguns casos, em outros não. Implemente `equals` explícito.
- VO carregando lógica de I/O ("Email::send"). VOs **não fazem nada com o mundo**.
- Comparação caso-sensitivo onde deveria ser insensitivo (`Email`).

## Links relacionados

- [[OOP-Moderno/Readonly-Classes]]
- [[OOP-Moderno/Property-Hooks]]
- [[Aggregates]]

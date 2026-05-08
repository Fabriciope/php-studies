---
tags: [php, oop, imutabilidade]
fase: 2
status: stub
---

# Readonly Classes

## Conceito

Em PHP 8.1 propriedades podem ser `readonly`: só atribuíveis no construtor (ou na própria classe), depois imutáveis. Em PHP 8.2 a anotação subiu para a classe inteira: `final readonly class Foo {}` torna **todas** as propriedades readonly automaticamente.

## Por que importa

Imutabilidade é a base de Value Objects e DTOs. Sem readonly, qualquer caller pode mutar `$user->email` e quebrar invariantes a posteriori. Com readonly, mutar é erro de compilação — o invariante vira propriedade da classe, não disciplina do time.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

final readonly class Money {
    public function __construct(
        public int $cents,
        public string $currency,
    ) {
        if ($cents < 0) {
            throw new InvalidArgumentException('cents must be >= 0');
        }
        if (strlen($currency) !== 3) {
            throw new InvalidArgumentException('currency must be ISO-4217');
        }
    }

    // mutação retorna NOVA instância
    public function add(self $other): self {
        if ($this->currency !== $other->currency) {
            throw new DomainException('currency mismatch');
        }
        return new self($this->cents + $other->cents, $this->currency);
    }
}

$price = new Money(1990, 'BRL');
// $price->cents = 0; // Error: Cannot modify readonly property
$total = $price->add(new Money(500, 'BRL'));
```

## Quando usar / quando não usar

- **Value Objects** — sempre.
- **DTOs** vindos da request, eventos, comandos — sempre.
- **Entidades de domínio com identidade** — depende. Se o estado muda (ex.: `Order` com status), readonly **na entidade** força reconstrução; muitas vezes preferível readonly nas propriedades imutáveis e mutabilidade controlada nas que mudam.
- **Não usar** se a classe precisa de hidratação por ORM via reflection mutável (Doctrine clássico) — embora ORMs modernos já lidem com readonly.

## Armadilhas comuns

- `clone` de objeto readonly **só funciona com `__clone`** que reatribui (PHP 8.3+ permite reatribuir readonly em `__clone`).
- Readonly não impede mutação de objeto **dentro** da propriedade: `readonly array` é "imutável" (arrays são copy-on-write), mas `readonly Cart` permite chamar `$cart->add($item)` se Cart não for ela mesma readonly.
- Serialização: `__construct` é chamado na desserialização? **Não** — readonly via `unserialize` ignora visibilidade. Cuidado com input não confiável.

## Links relacionados

- [[Asymmetric-Visibility]]
- [[Property-Hooks]]
- [[Arquitetura/Value-Objects]]

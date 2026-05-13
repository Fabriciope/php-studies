---
tags: [php, arquitetura, ddd, value-object]
fase: 4
status: draft
---

# Value Objects

## Conceito

O conceito de *Value Object* é central em **Eric Evans** (*DDD*, 2003, cap. 5) e reforçado por **Vaughn Vernon** (*Implementing DDD*, 2013, cap. 6, "*Value Objects*"). A definição de Evans: "um objeto que descreve uma característica de algo, sem identidade conceitual". Vernon endossa e adiciona a heurística "*prefira modelar como Value Object antes de modelar como Entity*" — a maior parte do que parece entidade no início é, na verdade, valor.

Três propriedades caracterizam um VO:

1. **Sem identidade.** Dois VOs com os mesmos atributos são o mesmo valor. `new Money(1990, 'BRL')` é igual a outro `new Money(1990, 'BRL')`. Em entidades, mesmo com atributos iguais, identidade difere (`Order#42` ≠ `Order#43`).
2. **Imutável.** Toda operação que pareceria "modificar" devolve **nova instância**. Em PHP 8.2+ isso é expresso com `final readonly class`.
3. **Validado na construção.** O construtor garante que **não existe instância inválida**. Princípio "*Always Valid*" — se você tem um `Email`, está validado; não há "talvez seja válido".

Outra propriedade derivada, defendida explicitamente por Vernon: **side-effect-free functions**. Métodos de um VO não fazem I/O, não chamam banco, não enviam e-mail. Devolvem ou novos VOs, ou tipos primitivos. Isso torna VOs **trivialmente testáveis** e **referencialmente transparentes** — chamar o mesmo método com os mesmos inputs sempre retorna o mesmo output.

```
      ┌──────────────────────────────────────────────┐
      │            Value Object                      │
      │                                              │
      │  - sem identidade (igualdade estrutural)     │
      │  - imutável (readonly)                       │
      │  - validado no construtor (always valid)     │
      │  - puro (sem I/O, sem efeito colateral)      │
      │  - métodos retornam novos VOs ou primitivos  │
      │                                              │
      └──────────────────────────────────────────────┘
```

O cheiro inverso, que VOs combatem, é o que Martin Fowler chama de **"Primitive Obsession"**: parâmetros e propriedades com tipos primitivos (`string $email`, `int $cents`, `string $cpf`) que **aceitam qualquer coisa**. Toda função que recebe `string $email` precisa revalidar; toda função que recebe `int $cents` pode receber `-5`. Promover esses primitivos a VOs **move a validação para o tipo** — não existe `Email` inválido em circulação.

VOs também são **combináveis**. `Money` + `Money` = `Money` (mesma moeda); `DateRange` operando com `DateTimeImmutable`. A linguagem do domínio se torna composicional, não procedural.

A relação com `enum` (PHP 8.1+) é direta: um enum de backed cases (ex: `OrderStatus`) **é** um VO degenerado (valores discretos finitos). VOs e enums coexistem — VO carrega valor contínuo (Money, Email, DateRange), enum carrega valor discreto fechado.

## Por que importa

O retorno cresce com o tamanho do sistema:

1. **Validação fica em um lugar só.** `Email` valida no construtor. Toda função que recebe `Email` confia. Em vez de 30 validações espalhadas, uma. Quando a regra muda (passa a aceitar `+` no local-part), você muda em um lugar.

2. **Erros de domínio em compile-time/static-analysis.** Se um método pede `Money`, você não passa `int` por engano. PHPStan/Psalm reclamam. Sem VO, `chargeOrder(int $amount)` aceita `chargeOrder(getOrderTotal())` onde `getOrderTotal()` retorna centavos vs reais — bug que só aparece em produção.

3. **Domínio expresso na assinatura.** `transferMoney(Money $amount, Account $from, Account $to)` é auto-documentado. `transferMoney(int $amount, int $fromId, int $toId)` exige docblock.

4. **Equality consistente.** Com VOs implementando `equals()`, comparações de domínio têm semântica clara. `Email::equals` pode ser case-insensitive; `CPF::equals` ignora pontuação. Sem VO, cada local compara do seu jeito.

5. **Refactor seguro.** Mudar `Money` de `int cents` para `decimal` (string interna) afeta uma classe. Sem VO, milhares de operações aritméticas precisam revisão.

Em 2-3 anos, esses ganhos são a diferença entre código de domínio que envelhece bem e código que vira pântano.

## Exemplo de código PHP

### 1. Email — validação no construtor + igualdade case-insensitive

```php
<?php
declare(strict_types=1);

namespace App\Domain\Shared;

final readonly class Email
{
    public function __construct(public string $address)
    {
        // Always Valid: não existe Email inválido em circulação
        if (!filter_var($address, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException("invalid email: {$address}");
        }
    }

    public function domain(): string
    {
        return substr($this->address, (int) strpos($this->address, '@') + 1);
    }

    public function local(): string
    {
        return substr($this->address, 0, (int) strpos($this->address, '@'));
    }

    // Igualdade estrutural explícita — case-insensitive por regra de domínio
    public function equals(self $other): bool
    {
        return strtolower($this->address) === strtolower($other->address);
    }

    public function __toString(): string
    {
        return $this->address;
    }
}
```

### 2. Money — operações que retornam nova instância (side-effect-free)

```php
<?php
declare(strict_types=1);

namespace App\Domain\Shared;

final readonly class Money
{
    public function __construct(
        public int $cents,
        public string $currency,
    ) {
        if ($cents < 0) {
            throw new \InvalidArgumentException('negative amount');
        }
        if (!preg_match('/^[A-Z]{3}$/', $currency)) {
            throw new \InvalidArgumentException('invalid ISO 4217 currency');
        }
    }

    public function add(self $other): self
    {
        $this->assertSameCurrency($other);
        return new self($this->cents + $other->cents, $this->currency);
    }

    public function subtract(self $other): self
    {
        $this->assertSameCurrency($other);
        if ($this->cents < $other->cents) {
            throw new \DomainException('result would be negative');
        }
        return new self($this->cents - $other->cents, $this->currency);
    }

    public function multiply(int $factor): self
    {
        if ($factor < 0) {
            throw new \InvalidArgumentException('negative factor');
        }
        return new self($this->cents * $factor, $this->currency);
    }

    public function isZero(): bool
    {
        return $this->cents === 0;
    }

    public function equals(self $other): bool
    {
        return $this->cents === $other->cents && $this->currency === $other->currency;
    }

    private function assertSameCurrency(self $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new \DomainException(
                "currency mismatch: {$this->currency} vs {$other->currency}"
            );
        }
    }
}
```

### 3. DateRange — VO compõe primitivos com regra

```php
<?php
declare(strict_types=1);

namespace App\Domain\Shared;

final readonly class DateRange
{
    public function __construct(
        public \DateTimeImmutable $from,
        public \DateTimeImmutable $to,
    ) {
        if ($to < $from) {
            throw new \InvalidArgumentException('to before from');
        }
    }

    public function contains(\DateTimeImmutable $when): bool
    {
        return $when >= $this->from && $when <= $this->to;
    }

    public function overlaps(self $other): bool
    {
        return $this->from <= $other->to && $other->from <= $this->to;
    }

    public function days(): int
    {
        return (int) $this->from->diff($this->to)->days;
    }

    public function equals(self $other): bool
    {
        return $this->from == $other->from && $this->to == $other->to;
    }
}
```

### 4. CPF — VO específico de domínio brasileiro

```php
<?php
declare(strict_types=1);

namespace App\Domain\Shared;

final readonly class Cpf
{
    public string $value;  // normalizado: só dígitos

    public function __construct(string $raw)
    {
        $digits = preg_replace('/\D/', '', $raw) ?? '';

        if (strlen($digits) !== 11 || !self::isValidCheckDigits($digits)) {
            throw new \InvalidArgumentException("invalid CPF: {$raw}");
        }

        $this->value = $digits;
    }

    public function format(): string
    {
        return sprintf(
            '%s.%s.%s-%s',
            substr($this->value, 0, 3),
            substr($this->value, 3, 3),
            substr($this->value, 6, 3),
            substr($this->value, 9, 2),
        );
    }

    public function equals(self $other): bool
    {
        return $this->value === $other->value;
    }

    private static function isValidCheckDigits(string $digits): bool
    {
        // implementação omitida — verifica os dígitos verificadores
        return true;
    }
}
```

### 5. VO + Enum combinados

```php
<?php
declare(strict_types=1);

namespace App\Domain\Order;

// Enum: VO degenerado (valores discretos)
enum OrderStatus: string
{
    case Pending = 'pending';
    case Paid = 'paid';
    case Shipped = 'shipped';
    case Cancelled = 'cancelled';

    public function canTransitionTo(self $next): bool
    {
        return match ([$this, $next]) {
            [self::Pending, self::Paid],
            [self::Pending, self::Cancelled],
            [self::Paid, self::Shipped],
            [self::Paid, self::Cancelled] => true,
            default => false,
        };
    }
}

// VO carrega status + timestamp da transição
final readonly class OrderStatusChange
{
    public function __construct(
        public OrderStatus $status,
        public \DateTimeImmutable $changedAt,
        public ?string $reason = null,
    ) {}
}
```

## Quando usar / quando não usar

- **Use** sempre que um conjunto de primitivas representa **um conceito** do domínio: Money, Email, CPF, DateRange, GeoPoint, OrderId, PhoneNumber.
- **Use** sempre que há regra associada a um valor (validação, normalização, operações).
- **Use** para IDs de agregados (`OrderId`, `CustomerId`) — UUIDs `string` perdem força de tipo.
- **Não use** para valores descartáveis sem regra (loop counter, índice de array temporário).
- **Não use** para strings genuinamente livres (corpo de comentário do usuário, conteúdo de post). Um "VO" `CommentBody` sem regra é overhead.
- **Não promova** primitivo a VO de forma reflexa em todo lugar — só onde há ganho (regra, validação, semântica).

## Armadilhas comuns

- **VO mutável.** Setter ou propriedade pública não-readonly. Perdeu o sentido — duas referências divergem. `final readonly class` é a forma canônica.
- **`equals()` ausente.** Comparar com `==` funciona às vezes em PHP mas é frágil (depende de propriedades não estarem em ordem diferente, de tipos exatos). Sempre implemente `equals(self $other): bool` explícito.
- **VO fazendo I/O.** `Email::send()`, `Cpf::queryReceita()`. VOs são **puros**. Send é responsabilidade de um *Mailer*; query de receita é de um *Service*. Mantenha VO sem efeito colateral.
- **Validação fraca no construtor.** Aceitar `Email` que só checa `strpos('@')`. Use `filter_var` ou similar — validação fraca é pior que sem validação porque dá falsa segurança.
- **Comparação case/format-sensitive errada.** `Email` comparado case-sensitive (`a@x.com` ≠ `A@x.com`), `Cpf` comparado com pontuação diferente. Normalize ou compare normalizado.
- **VO gigante.** Um VO com 15 campos sinaliza que ali tem uma entidade ou mais de um VO misturados. Decomponha.
- **VO escondendo entidade.** Se há identidade (dois "iguais" precisam ser distintos por ID), não é VO — é entidade. `Person` raramente é VO; `Address` quase sempre é.
- **VO acoplado a framework.** `Email` extendendo `Symfony\Component\Validator\Constraint`. Vaza framework para o domínio. Deixe VO independente; valide em camada apropriada se precisar de integração com framework.

## Links relacionados

- [[OOP-Moderno/Readonly-Classes]]
- [[OOP-Moderno/Property-Hooks]]
- [[Aggregates]]
- [[Use-Cases]]
- [[Camadas-DDD]]
- [[Domain-Events-CQRS]]

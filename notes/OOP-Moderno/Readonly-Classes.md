---
tags: [php, oop, imutabilidade]
fase: 2
status: draft
---

# Readonly Classes

## Conceito

PHP 8.1 introduziu o modificador `readonly` em **propriedades** via RFC "Readonly properties 2.0": uma vez inicializadas (no construtor ou em escopo da própria classe), nunca mais podem ser reatribuídas. Qualquer tentativa de mutação externa, mesmo via reflection sem flag específica, lança `Error`. O modificador aplica-se a propriedades **tipadas** — sem type, o PHP não consegue garantir a inicialização única.

PHP 8.2 elevou a abstração com a RFC "Readonly classes": `readonly class Foo {}` marca **todas** as propriedades declaradas (e promovidas) como readonly. A classe inteira passa a ser, por declaração, um conjunto fechado de campos imutáveis. Dinâmicas (`$this->novo = 1`) já eram proibidas em classes com `dynamic properties` desativadas; em readonly class são proibidas categoricamente.

PHP 8.3 fechou o gap mais doloroso: `clone` de objeto readonly. Antes, copiar e ajustar um campo exigia recriar via construtor (`new self($this->a, $novoB)`), o que falhava em hierarquias com muitos campos. A RFC "Readonly amendments" permite **reatribuir propriedades readonly dentro do método mágico `__clone`** — durante a janela do clone, as propriedades voltam a ser escrevíveis exclusivamente naquele escopo. Daí o idioma `with*()`: `clone $this` seguido de reatribuição.

O modelo mental certo: readonly em PHP é **shallow**. A referência ao objeto é fixa, mas o objeto referenciado pode ser mutável. `readonly Cart $cart` impede trocar o `$cart` por outro, não impede `$cart->add($item)`. Para imutabilidade profunda, **toda a cadeia** precisa ser readonly (ou Value Object). Isso difere de `const` em C++ ou `val` em Kotlin com tipos imutáveis — readonly não se propaga.

Compare também com `final class`: `final` impede herança; `readonly` impede mutação. São ortogonais. Value Objects canônicos costumam ser `final readonly class` — fecham herança (não faz sentido `MoneyEspecial extends Money`) e mutação (mudar dá outro objeto).

## Por que importa

**Cenário 1 — invariantes garantidos por construção**: um `Money(1990, 'BRL')` valida no construtor e nunca mais pode entrar em estado inválido. Sem readonly, qualquer caller pode `$money->cents = -1` depois e violar a invariante; o teste falha longe da origem. Com readonly, o invariante é estrutural, não cultural.

**Cenário 2 — concorrência e cache**: objetos imutáveis são naturalmente thread-safe (no PHP, isso vale para extensões como Swoole/parallel) e seguros para cache em memória — não há como um consumidor mutar e contaminar outro. Sem readonly, precisaria clonar defensivamente.

**Cenário 3 — DTOs de borda**: comandos, eventos, requisições. O DTO chega do controlador, percorre handlers, é serializado em logs. Se algum middleware mutar, o log mente. Readonly assegura que o que entrou é o que saiu.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// 1) Value Object canônico: final readonly class.
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

    // Operação retorna NOVA instância — nunca muta self.
    public function add(self $other): self {
        if ($this->currency !== $other->currency) {
            throw new DomainException('currency mismatch');
        }
        return new self($this->cents + $other->cents, $this->currency);
    }

    public function multiply(int $factor): self {
        return new self($this->cents * $factor, $this->currency);
    }
}

$preco = new Money(1990, 'BRL');
// $preco->cents = 0;                     // Error: Cannot modify readonly property
$total = $preco->add(new Money(500, 'BRL'));
```

```php
<?php
declare(strict_types=1);

// 2) Readonly em propriedade individual (8.1) vs classe inteira (8.2).
final class User8_1 {
    public function __construct(
        public readonly string $id,
        public readonly string $email,
        public int $loginCount = 0,    // MUTÁVEL — não tem readonly.
    ) {}
}

final readonly class User8_2 {
    public function __construct(
        public string $id,
        public string $email,
        public int $loginCount = 0,    // TAMBÉM readonly — herdou da classe.
    ) {}
}
// 8.2: não há como ter propriedade mutável em readonly class.
// Se precisa de mistura, fique em 8.1 com readonly por propriedade.
```

```php
<?php
declare(strict_types=1);

// 3) PHP 8.3: clone-with via __clone.
final readonly class Address {
    public function __construct(
        public string $street,
        public string $city,
        public string $zip,
    ) {}

    // O idioma `with*` para imutáveis.
    public function withCity(string $city): self {
        $copy = clone $this;
        // Reatribuição permitida apenas dentro de __clone, mas chamada aqui
        // funciona porque o PHP 8.3 abriu a janela durante o clone.
        // Para PHP < 8.3, use: return new self($this->street, $city, $this->zip);
        return $copy->withMutation(fn($c) => $c->city = $city);
    }

    // Helper genérico: aplica mutação dentro do __clone window.
    private function withMutation(\Closure $fn): self {
        $clone = clone $this;
        $clone->__cloneApply($fn);
        return $clone;
    }

    public function __clone(): void {
        // Janela 8.3: aqui podemos reatribuir readonly.
    }

    private function __cloneApply(\Closure $fn): void {
        // Em 8.3+ esta linha funciona dentro da janela de clone.
        $fn(\Closure::bind(fn() => $this, $this, self::class)());
    }
}

// Forma idiomática mais simples em 8.3:
final readonly class Point {
    public function __construct(public int $x, public int $y) {}

    public function withX(int $x): self {
        $new = clone $this;
        // PHP 8.3+: reatribuição permitida via __clone hook.
        (function() use ($x) { $this->x = $x; })->call($new);
        return $new;
    }
}
```

```php
<?php
declare(strict_types=1);

// 4) Pegadinha: readonly é shallow.
final readonly class Order {
    /** @param list<string> $tags */
    public function __construct(
        public string $id,
        public array $tags,           // array é copy-on-write — seguro.
        public Customer $customer,    // objeto NÃO é readonly por tabela.
    ) {}
}

$o = new Order('o1', ['vip'], $customer);
// $o->tags[] = 'novo';        // Error: indirect modification of readonly.
// $o->customer = $outro;      // Error: cannot modify readonly property.
$o->customer->rename('Novo');  // OK — Customer não é readonly. Mutou!
// Solução: Customer também precisa ser readonly para imutabilidade profunda.
```

```php
<?php
declare(strict_types=1);

// 5) Hidratação via reflection (ORMs, deserializers).
// Doctrine 3.x e Symfony Serializer suportam readonly via API específica
// (ReflectionProperty::setValue mesmo em readonly), mas há custo de performance.
$ref = new \ReflectionClass(Money::class);
$obj = $ref->newInstanceWithoutConstructor();      // pula __construct
$prop = $ref->getProperty('cents');
$prop->setValue($obj, 1500);                       // funciona em readonly
$prop = $ref->getProperty('currency');
$prop->setValue($obj, 'USD');
// Útil para deserializers; perigoso porque pula validação do construtor.
```

## Quando usar / quando não usar

- **Value Objects**: sempre `final readonly class`. Identidade por valor + sem herança + imutável é o pacote completo.
- **DTOs de borda** (Command, Query, Event, Request): sempre. Garantem que o payload não muta entre camadas.
- **Configurações imutáveis** carregadas no boot: readonly evita drift acidental durante a request.
- **Entidades de domínio com identidade**: depende. Use readonly nas propriedades que nunca mudam (`id`, `createdAt`) via 8.1; deixe mutáveis as que mudam por design (`status`, `updatedAt`).
- **Não use** se a classe precisa de hidratação por ORM que ainda não suporta readonly (Doctrine ≤ 2.13, alguns serializers antigos).
- **Não use** quando o objeto representa estado que evolui in-place por motivos de performance (caches, buffers, builders).
- **Não use** se você simplesmente quer "esconder o setter" — para isso, prefira [[Asymmetric-Visibility]] (`private(set)`).

## Armadilhas comuns

- **Shallow readonly**: `readonly Cart $cart` não impede `$cart->add(...)`. Acontece porque o readonly protege a *referência*, não o grafo. Solução: tornar `Cart` também readonly ou usar collections imutáveis.
- **`clone` sem PHP 8.3 falha em readonly classes**: até 8.2, `clone` cria cópia mas você não pode reatribuir nenhum campo da cópia. Em 8.3 a janela abriu via `__clone`. Verifique a versão antes de adotar o padrão `with*`.
- **`unserialize` ignora visibilidade e construtor**: dado serializado de origem não confiável pode reconstruir objeto readonly em estado inválido, pulando validações. Mitigue com `__unserialize`/`__wakeup` ou allowed_classes.
- **Tipos `mixed` ou sem tipo não aceitam readonly**: o PHP exige tipo declarado. É proposital — sem tipo, o motor não consegue garantir inicialização única.
- **Promoção readonly exige PHP 8.1+**: combinar `public readonly` em constructor promotion só funciona da 8.1 em diante. Em 8.0, você teve promotion sem readonly.
- **`new self(...)` em hierarquia retorna o tipo concreto errado**: em classe readonly não-final, `static` (LSB) e `self` divergem. Em VO sempre marque `final` para evitar surpresa.
- **Mutar array readonly é "indirect modification"**: `$obj->list[] = 'x'` falha mesmo sendo um array. O PHP trata como modificação. Para append, crie novo array e reatribua via clone-with.
- **Reflection ainda pode reatribuir**: ferramentas de teste/serialização usam `ReflectionProperty::setValue` para contornar. Não confie em readonly como barreira de segurança — é barreira de design.

## Links relacionados

- [[Asymmetric-Visibility]]
- [[Property-Hooks]]
- [[Arquitetura/Value-Objects]]
- [[Constructor-Promotion]]
- [[Enums]]
- [[Sistema-de-Tipos]]

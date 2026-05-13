---
tags: [php, oop, php84]
fase: 2
status: draft
---

# Asymmetric Visibility (PHP 8.4)

## Conceito

Asymmetric visibility, introduzida pela RFC "Asymmetric Visibility v2" no PHP 8.4, permite declarar visibilidades **distintas para leitura e escrita** de uma mesma propriedade. A leitura usa o modificador principal; a escrita usa um modificador entre parênteses sufixado por `(set)`. Combinações comuns: `public private(set)`, `public protected(set)`, `protected private(set)`.

O modelo mental: **a propriedade tem duas portas**. Quem chega pela porta de leitura segue uma regra; quem chega pela porta de escrita segue outra. Antes de 8.4, a única forma de obter esse comportamento era declarar a propriedade `private` (ou `protected`) e expor um getter público — duas linhas de boilerplate, sem ganho semântico, e a propriedade ficava "escondida" do tipo. Com asymmetric, a propriedade continua sendo propriedade — aparece em reflection, type hint, autocomplete — mas com semântica de acesso refinada.

A regra estrutural: a visibilidade de `set` precisa ser **igual ou mais restritiva** que a de `get`. `public private(set)` é válido (leitura mais aberta que escrita); o inverso `private public(set)` é erro de compilação — não faria sentido permitir escrever de fora e proibir ler. Para "escrever mais aberto que ler" você precisaria virar a relação: o tipo é `private`-by-default-de-leitura mas escrita pública? Não há caso de uso, e o PHP proíbe.

Asymmetric **combina** com promotion de construtor, readonly, property hooks e tipos. O construtor (ou métodos da própria classe) escrevem pela visibilidade mais restritiva; o mundo externo lê pela aberta. Promotion + asymmetric é a combinação canônica para entidades com identidade controlada: `public function __construct(public private(set) string $id) {}`.

Comparado a readonly: readonly é estritamente mais forte porque proíbe **qualquer** escrita após inicialização, inclusive de dentro da classe (exceto construtor e `__clone` em 8.3+). Asymmetric permite a classe continuar mutando livremente; só veda mutação **externa**. Os dois resolvem problemas diferentes — readonly = imutabilidade; asymmetric = encapsulamento direcional.

Performance: idêntica a propriedade comum. Diferente de hooks, asymmetric é verificação de visibilidade em compile-time/JIT, sem chamada de função. Não há overhead em runtime.

## Por que importa

**Cenário 1 — entidade com ID gerado**: `Order` precisa expor `id` para qualquer leitor, mas só a própria classe (ou fábrica) deve atribuir. Sem asymmetric, ou você expõe setter público (perigoso) ou esconde com getter (boilerplate). Com `public private(set) string $id`, a propriedade é honesta: lê todo mundo, escreve só dentro.

**Cenário 2 — propriedade que muda controladamente**: contadores, status. `$pedido->status` deve ser legível em qualquer lugar, mas só métodos de domínio (`markAsPaid()`) podem mudar. `public private(set) OrderStatus $status` casa exatamente — readonly não serve porque o estado *muda*; getter/setter atrapalha porque expõe escrita.

**Cenário 3 — herança com escrita protegida**: classe base define `protected(set) string $tableName` para subclasses configurarem. Antes você usaria método abstrato `getTable()`; agora é propriedade declarativa que o IDE entende.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// 1) Básico: leitura pública, escrita privada.
final class Counter {
    public private(set) int $value = 0;

    public function increment(): void {
        $this->value++;          // OK: estamos dentro da classe.
    }

    public function reset(): void {
        $this->value = 0;        // OK.
    }
}

$c = new Counter();
echo $c->value;        // 0  — leitura pública
$c->increment();
echo $c->value;        // 1
// $c->value = 99;     // Error: Cannot modify private(set) property from global scope.
```

```php
<?php
declare(strict_types=1);

// 2) Combinado com constructor promotion — uso idiomático em entidades.
final class Order {
    public function __construct(
        public private(set) string $id,
        public private(set) OrderStatus $status,
        public readonly DateTimeImmutable $createdAt,
    ) {}

    // Transição de estado controlada.
    public function markAsPaid(): void {
        if ($this->status !== OrderStatus::Pending) {
            throw new DomainException('cannot pay non-pending order');
        }
        $this->status = OrderStatus::Paid;
    }
}

$o = new Order('o1', OrderStatus::Pending, new DateTimeImmutable());
echo $o->status->value;     // pending — leitura externa ok
$o->markAsPaid();           // mutação interna controlada
// $o->status = OrderStatus::Cancelled;  // Error.
// $o->createdAt = new DateTimeImmutable();  // Error: readonly.
```

```php
<?php
declare(strict_types=1);

// 3) Herança: protected(set) abre escrita para subclasses.
abstract class Repository {
    public protected(set) string $tableName = '';

    public function find(int $id): ?array {
        // usa $this->tableName em SQL.
        return null;
    }
}

final class UserRepository extends Repository {
    public function __construct() {
        $this->tableName = 'users';     // OK: subclasse pode escrever.
    }
}

$repo = new UserRepository();
echo $repo->tableName;  // 'users' — leitura pública.
// $repo->tableName = 'x';  // Error: protected(set), fora da hierarquia.
```

```php
<?php
declare(strict_types=1);

// 4) Asymmetric + Property Hooks: combinação que valida sem expor escrita.
final class Email {
    public private(set) string $value {
        set(string $v) {
            $v = strtolower(trim($v));
            if (!filter_var($v, FILTER_VALIDATE_EMAIL)) {
                throw new InvalidArgumentException('invalid email');
            }
            $this->value = $v;
        }
    }

    public function __construct(string $value) {
        $this->value = $value;     // dispara hook + permitido pelo set privado.
    }

    public function changeTo(string $value): void {
        $this->value = $value;     // dentro da classe — OK.
    }
}

$e = new Email('Foo@Bar.com');
echo $e->value;     // foo@bar.com
// $e->value = 'x@y.com'; // Error de visibilidade — nem chega ao hook.
```

```php
<?php
declare(strict_types=1);

// 5) Quando NÃO substitui readonly.
final class Imutavel {
    public private(set) string $id;       // muda dentro, externo só lê.

    public function __construct(string $id) {
        $this->id = $id;
    }

    public function reassign(string $newId): void {
        $this->id = $newId;     // PERMITIDO — viola "imutabilidade" do objeto.
    }
}

// Para garantir que nem a própria classe muda após construção:
final class RealmenteImutavel {
    public function __construct(
        public readonly string $id,   // só __construct (e __clone em 8.3+).
    ) {}
}
// Conclusão: asymmetric controla QUEM, readonly controla QUANDO.
```

## Quando usar / quando não usar

- **Use `public private(set)`** para propriedades que mudam ao longo da vida do objeto **apenas via métodos da própria classe** (contadores, status, flags).
- **Use `public protected(set)`** quando subclasses precisam configurar/mutar mas o mundo externo não.
- **Combine com promotion** em entidades de domínio: declaração + visibilidade + atribuição em uma linha.
- **Combine com hooks** quando a escrita interna ainda precisa validar.
- **Não substitua readonly**: se a propriedade nunca muda após construção, readonly é semanticamente mais forte e comunica melhor a intenção.
- **Não use para encapsular "tudo private + getter"**: se o getter teria lógica não trivial, o que você quer é método ou property hook, não asymmetric.
- **Não use** quando a propriedade tem cálculo derivado — vire virtual property com `get =>`.

## Armadilhas comuns

- **Visibilidade de `set` mais aberta que `get` é erro**: `private public(set)` não compila. Acontece porque o PHP exige consistência — quem pode escrever logicamente também pode ler.
- **Confundir com readonly**: programadores acham que `public private(set)` "torna imutável". Não torna — a própria classe ainda muda. Para imutabilidade real, use `readonly`.
- **Reflection ignora restrição**: `ReflectionProperty::setValue` consegue escrever ignorando asymmetric (assim como ignora `private`). Não é barreira de segurança, é barreira de design.
- **Asymmetric + readonly é redundante**: `public private(set) readonly` colapsa para readonly — `readonly` já implica que só `__construct` escreve. O motor aceita mas a parte `private(set)` não agrega.
- **Clone e visibilidade**: `clone` cria uma cópia que respeita as mesmas regras de visibilidade. Não há "modo de clone" que abra a escrita externa.
- **Mensagens de erro genéricas**: tentar escrever de fora gera `Error: Cannot modify private(set) property`. Em código que envolve trait/closure rebound, pode confundir sobre qual escopo está realmente "fora".
- **Closures e visibilidade**: `Closure::bind` para escopo da classe consegue escrever — mesmo comportamento de `private` normal. Útil em testes, perigoso em código de produção.
- **Promovida + asymmetric exige PHP 8.4**: combinar `public private(set)` em promotion não existe em versões anteriores. Verifique runtime antes de adotar.

## Links relacionados

- [[Readonly-Classes]]
- [[Property-Hooks]]
- [[Constructor-Promotion]]
- [[Arquitetura/Value-Objects]]
- [[Enums]]
- [[Sistema-de-Tipos]]

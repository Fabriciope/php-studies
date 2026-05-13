---
tags: [php, fundamentos, tipos]
fase: 1
status: draft
---

# Sistema de Tipos

## Conceito

O sistema de tipos do PHP evoluiu de "type hinting" opcional (PHP 5) para um sistema rico de tipos compostos a partir do PHP 7-8. Hoje cobre escalares (`int`, `float`, `string`, `bool`), composto (`array`, `iterable`, `object`, `callable`), classes/interfaces, e tipos especiais (`void`, `never`, `mixed`, `self`, `static`, `parent`, `null`, `true`, `false`).

O salto qualitativo veio em duas RFCs: **Union Types v2 (PHP 8.0)** permitiu `A|B`; **Pure Intersection Types (PHP 8.1)** introduziu `A&B`; **DNF Types (PHP 8.2)** liberou a forma normal disjuntiva `(A&B)|C`, único modo legal de misturar união e interseção. Antes de 8.2, `(Logger&Tagger)|null` era inválido — agora é a forma canônica para "interseção opcional".

O modelo mental: o sistema é **nominal** para classes/interfaces (não estrutural como TypeScript) e **estrutural** apenas para `callable` e `iterable`. Isso significa que duas classes com a mesma forma não são intercambiáveis — só implementação explícita de interface conta. PHP **não** tem generics como tipos de primeira classe (`array<int, User>` existe só em PHPDoc, lido por Psalm/PHPStan, ignorado em runtime).

**Variância**: PHP segue Liskov com regras explícitas desde 7.4. Tipos de **retorno são covariantes** (subclasse pode retornar tipo mais específico). Tipos de **parâmetro são contravariantes** (subclasse pode aceitar tipo mais amplo). Na prática, contravariância de parâmetro é raramente usada porque exige declarar pai com tipo amplo (`mixed`/`object`); o caso comum é covariância de retorno em factory methods.

**Tipos especiais**:
- `void` — função não retorna valor (não pode ter `return $x;`, pode ter `return;`).
- `never` — função **nunca** retorna normalmente (sempre lança ou loopa infinito). Tipo bottom, subtipo de tudo. Permite a análise estática eliminar branches.
- `mixed` — top type, equivalente a "sem tipo", mas explícito. Subtipos: todos. Usado quando genuinamente qualquer valor é aceitável.
- `static` — late static binding como tipo. Em retorno, força subclasses a retornar instância da própria classe (não da declarante).
- `self` — referência à classe que declara (early binding). Diferente de `static` em herança.
- `true`, `false`, `null` — tipos **literais**. `bool` = `true|false`. Útil em union: `function find(...): User|false`.

**Comparação com TypeScript**: PHP carece de generics genuínos, tipos literais arbitrários (`'admin'|'user'`), tipos condicionais, mapped types e narrowing por type guard. Em compensação, runtime checks são reais e `TypeError` é determinístico. PHPStan/Psalm cobrem boa parte dos gaps via PHPDoc.

**Resolução de tipos é em runtime**: ao contrário de Java/C# que apagam (erasure) ou reificam genéricos no bytecode, PHP **carrega cada tipo no metadata da função** e checa em cada chamada. Isso tem custo, mas é o que garante `TypeError` determinístico. O custo é compensado pelo OPcache ao cachear o resultado da análise para chamadas seguintes.

**Tipos não impedem `null`**: por padrão, `function f(int $x)` **não** aceita null. Para aceitar, escreva `?int` (sugar para `int|null`) ou `int|null` literal. A diferença: `?int $x = null` torna o parâmetro opcional; `int|null $x` ainda exige passagem explícita. A partir de 8.1, omitir `?` mas default `= null` é deprecated — alinhe a sintaxe.

## Por que importa

**Cenário 1 — Result type sem exception abuse**: modelar `findUser(int $id): User|UserNotFound` documenta no retorno que falha é possível, força `match` no caller e elimina `try/catch` de fluxo de controle. Sem union types, ou retorna `?User` (perde-se causa) ou lança exception (caro, polui stack trace para caso esperado).

**Cenário 2 — composição de capacidades sem herança múltipla**: um handler precisa de `Logger&Tagger`. Sem intersection, ou cria interface combinada (acoplamento) ou recebe duas dependências (verbosidade). Intersection casa com Interface Segregation Principle sem inventar nome novo.

**Cenário 3 — funções que sempre lançam**: `function fail(string $m): never` permite que análise estática entenda que o código depois da chamada é **inalcançável**, eliminando falsos positivos sobre "variável possivelmente não inicializada" em branches.

**Cenário 4 — APIs legadas com booleano de falha**: `mysqli::query()` retorna `mysqli_result|false`. Tipar caller como `mysqli_result|false` força tratar `false` antes de usar. Sem tipos literais, esse contrato era opaco.

**Cenário 5 — variância em factories**: framework define `abstract class Repository { abstract public function findOne(): ?Entity; }`. Subclasse `UserRepository` quer retornar `?User`. Sem covariância de retorno, era impossível sem cast; com covariância, `: ?User` é override válido e o tipo permanece preciso para os consumidores.

## Exemplo de código PHP

**Union types — sucesso ou falha tipada:**

```php
<?php
declare(strict_types=1);

final class User { public function __construct(public readonly int $id) {} }
final class UserNotFound { public function __construct(public readonly int $id) {} }

function findUser(int $id): User|UserNotFound {
    return $id > 0 ? new User($id) : new UserNotFound($id);
}

$result = findUser(42);
match (true) {
    $result instanceof User         => print "achou {$result->id}",
    $result instanceof UserNotFound => print "perdeu {$result->id}",
};
```

**Intersection types — duas capacidades obrigatórias:**

```php
<?php
declare(strict_types=1);

interface Logger { public function log(string $msg): void; }
interface Tagger { public function tag(string $key, string $value): void; }

// Cada argumento precisa implementar AMBAS interfaces
function emit(Logger&Tagger $sink, string $event): void {
    $sink->tag('event', $event);
    $sink->log("emitted {$event}");
}

final class StdSink implements Logger, Tagger {
    public function log(string $msg): void { error_log($msg); }
    public function tag(string $k, string $v): void { /* ... */ }
}

emit(new StdSink(), 'order.created'); // OK
```

**DNF types (PHP 8.2) — interseção opcional:**

```php
<?php
declare(strict_types=1);

// (Logger&Tagger) OU null — DNF é a única forma válida
function tryEmit((Logger&Tagger)|null $sink, string $event): void {
    $sink?->tag('event', $event);
    $sink?->log($event);
}
```

**`never` em validator que sempre falha:**

```php
<?php
declare(strict_types=1);

function abort(string $reason): never {
    throw new RuntimeException($reason);
}

function divide(int $a, int $b): int {
    if ($b === 0) {
        abort('division by zero');
    }
    // análise estática sabe: $b !== 0 a partir daqui
    return intdiv($a, $b);
}
```

**Covariância de retorno (LSP em ação):**

```php
<?php
declare(strict_types=1);

abstract class Animal {}
final class Dog extends Animal {}

abstract class Shelter {
    abstract public function rescue(): Animal;
}

final class DogShelter extends Shelter {
    // retorno mais específico — permitido (covariância)
    public function rescue(): Dog {
        return new Dog();
    }
}
```

**`self` vs `static` (LSB tipado):**

```php
<?php
declare(strict_types=1);

class Builder {
    public function withName(string $n): static { // retorna o tipo CONCRETO
        $this->name = $n;
        return $this;
    }
}

final class ProductBuilder extends Builder {
    public function withSku(string $s): static { /* ... */ return $this; }
}

// `static` permite encadear:
(new ProductBuilder())->withName('x')->withSku('y'); // tipo inferido: ProductBuilder

// Se fosse `self`, withName retornaria Builder e perderia withSku.
```

**Anti-padrão — `mixed` como desistência:**

```php
<?php
declare(strict_types=1);

// RUIM — diz nada
function process(mixed $input): mixed { /* ... */ }

// MELHOR — diz o que é
function process(string|int|array $input): Order|OrderError { /* ... */ }
```

**Generics simulados via PHPDoc (lidos por PHPStan/Psalm):**

```php
<?php
declare(strict_types=1);

/**
 * @template T of object
 */
final class Collection {
    /** @var list<T> */
    private array $items = [];

    /** @param T $item */
    public function add(object $item): void {
        $this->items[] = $item;
    }

    /** @return list<T> */
    public function all(): array {
        return $this->items;
    }
}

/** @var Collection<User> $users */
$users = new Collection();
$users->add(new User(1));
foreach ($users->all() as $u) {
    // PHPStan sabe: $u é User
    echo $u->id;
}
```

Em runtime, é só `array`. Mas a análise estática enforça o tipo. Único caminho para generics em PHP hoje.

**Tabela de tipos especiais:**

| Tipo     | Posição válida          | Significado                                 |
|----------|-------------------------|---------------------------------------------|
| `void`   | retorno                 | Sem valor de retorno                        |
| `never`  | retorno                 | Nunca retorna (lança ou loopa)              |
| `mixed`  | param, retorno, prop    | Qualquer valor (top type)                   |
| `self`   | qualquer                | Classe declarante (early binding)           |
| `static` | retorno, param (8.0+)   | Late static binding                         |
| `null`   | em union                | Apenas `null`                               |
| `true`   | em union (8.2+)         | Apenas `true`                               |
| `false`  | em union                | Apenas `false`                              |
| `iterable` | qualquer              | `array \| Traversable`                      |

## Quando usar / quando não usar

- **Union `A|B`** quando há dois resultados legítimos e o caller deve tratar ambos. Especialmente sucesso/falha de domínio.
- **Intersection `A&B`** quando o parâmetro precisa de **duas capacidades** que não fazem sentido como uma interface única (segregar é o objetivo).
- **`never`** em funções de aborto, redirect HTTP, `throw` wrapper. Melhora narrowing em PHPStan/Psalm.
- **`static`** em fluent interfaces e factory methods. **`self`** quando explicitamente NÃO quer overriding (final, builder de classe abstrata fechada).
- **`mixed`** raramente: serializers, container DI genérico, dump/debug. Em domínio, é sinal de design incompleto.
- **`true`/`false` literais** em retornos de função booleana onde só um valor é esperado (parser que retorna `array|false`).
- **Não usar** generics em PHPDoc se não rodar Psalm/PHPStan — gera ilusão de tipo.
- **Não substituir** Enum por union de literais quando tem comportamento associado. Enum sempre vence.

## Armadilhas comuns

- **`void` ≠ `null`**: `void` proíbe `return $x;`; quem chama uma função `void` **não pode atribuir** o resultado (em strict PHPStan acusa). Causa: `void` é "ausência de retorno", `null` é "retorno cujo valor é null".
- **`self` em factory abstrata trava herança**: declarar `static function make(): self` na base faz subclasse retornar instância da base, não dela mesma. Use `static`. Causa: `self` é resolvido em tempo de compilação.
- **Union de classes desabilita `match` exaustivo**: `match` não sabe enumerar subtipos abertos; só Enum garante exaustividade. Causa: classes podem ser estendidas em runtime, Enum é fechado.
- **`iterable` aceita `Generator`**: se o caller fizer `count($arg)` em `iterable`, quebra para Generator (não é Countable). Causa: `iterable = array | Traversable`, e Traversable inclui Generator.
- **Interseção exige interfaces, não classes concretas**: `Foo&Bar` onde Foo é classe concreta é proibido. Causa: PHP não tem herança múltipla de classes; interseção só faz sentido entre contratos.
- **Covariância de retorno exige PHP 7.4+; antes, override quebrava**. Causa: variância foi adicionada explicitamente; antes o engine exigia tipo idêntico.
- **`null` em parâmetro typed sem `?` está deprecated em 8.1+**: `function f(int $x = null)` agora emite deprecation. Use `?int $x = null`. Causa: ambiguidade entre "tipo nullable" e "default null"; o RFC forçou explicitude.
- **`callable` é estrutural mas não checa assinatura**: `function f(callable $cb)` aceita qualquer coisa chamável, **sem** validar aridade ou tipos dos parâmetros. Use `Closure` + PHPDoc para mais rigor. Causa: PHP resolve callables tardiamente (strings, arrays, closures, invokables).
- **PHPDoc generic `array<int,User>` é ficção em runtime**: o engine ignora; só PHPStan/Psalm leem. Em runtime, é `array` e ponto. Causa: PHP não tem reified generics.
- **Intersection com `Stringable` e classe concreta que tem `__toString`**: a classe precisa **declarar** `implements Stringable` explicitamente em PHP 8.0+. Causa: sistema nominal — método `__toString` por si só não satisfaz interface.

## Links relacionados

- [[Strict-Types]]
- [[Hierarquia-Throwable]]
- [[OOP-Moderno/Enums]]
- [[OOP-Moderno/Readonly-Classes]]
- [[Padroes/SOLID]]

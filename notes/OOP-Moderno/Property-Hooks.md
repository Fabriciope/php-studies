---
tags: [php, oop, php84]
fase: 2
status: draft
---

# Property Hooks (PHP 8.4)

## Conceito

Property hooks, introduzidos pela RFC homônima do PHP 8.4, permitem interceptar a **leitura** (`get`) e a **escrita** (`set`) de uma propriedade sem expor um par getter/setter explícito. A sintaxe coloca os hooks entre chaves logo após a declaração da propriedade — o caller continua usando `$obj->prop`, mas a classe controla o que acontece em torno do acesso.

O modelo mental é o de **propriedades computadas** de C# (`public string Name { get; set; }`) ou Kotlin (`var name: String get() = ... set(value) { ... }`). PHP escolheu uma sintaxe próxima destas, mas com particularidades: dentro do `set` você pode atribuir a `$this->prop` para gravar no *backing field* (o storage real da propriedade) sem disparar recursão; o motor diferencia atribuição-de-storage de atribuição-via-hook.

Existem duas formas de hook. **Backed hook**: a propriedade tem storage de verdade; o hook executa antes/depois do acesso. Exemplo: `set { $this->email = strtolower($value); }`. **Virtual property**: a propriedade *não tem storage*; só hook `get` (e opcionalmente `set` que delega para outra fonte). É o caso de campos derivados: `get => $this->first . ' ' . $this->last`. Virtual properties não consomem memória da instância e não aparecem em `var_dump` como slot.

Hooks também podem ser declarados em **interfaces**, contratando que uma classe implementadora forneça `get` e/ou `set` para a propriedade. Isso resolve o problema histórico de "interface não pode exigir propriedade" — antes você caía em `getName(): string`. Agora a interface pode declarar `public string $name { get; set; }` e implementações podem ser propriedade pura, hook backed ou virtual.

Performance: hook engatilha chamada de método, então leitura/escrita custa mais que propriedade pura. A diferença é desprezível para acesso ocasional, mas relevante em loops apertados sobre milhões de objetos. Compare com C# JIT, que costuma inlinear getters/setters triviais — o PHP ainda não faz essa otimização agressivamente.

Hooks interagem com herança como métodos: subclasse pode sobrescrever hooks da parent. Combinam com `readonly` (apenas `get` faz sentido), com asymmetric visibility, e com type coercion das declarações da propriedade.

## Por que importa

**Cenário 1 — encapsular validação sem boilerplate**: você precisa garantir que `email` seja sempre normalizado. Sem hooks, ou expõe `public string $email` (e qualquer caller estraga) ou cria `setEmail(string)` + `getEmail(): string` e força chamada por método. Com hook `set`, a validação mora na propriedade e o caller usa `$user->email = 'X'` normalmente.

**Cenário 2 — propriedades derivadas leves**: `fullName` é `$first . ' ' . $last`. Antes, virava `getFullName(): string`. Com virtual property, vira `public string $fullName { get => ... }` — sem storage, expressivo, ainda type-safe.

**Cenário 3 — migração suave de propriedade pública para encapsulada**: você publicou `public int $count`. Anos depois descobre que precisa validar. Trocar para `private $count` + getter/setter quebra todos os call sites. Com hooks, adiciona `{ set { ... } }` sem mudar a API pública nem um caractere de call site.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// 1) Backed hook: storage real + validação no set.
final class User {
    public string $email {
        set(string $value) {
            $value = strtolower(trim($value));
            if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
                throw new InvalidArgumentException('invalid email');
            }
            // Atribuição ao próprio storage — NÃO recursiona.
            $this->email = $value;
        }
    }

    public function __construct(string $email) {
        $this->email = $email;   // dispara o set hook
    }
}

$u = new User('  Foo@Bar.COM ');
echo $u->email;   // foo@bar.com
$u->email = 'novo@x.com';   // hook valida e normaliza
```

```php
<?php
declare(strict_types=1);

// 2) Virtual property: sem storage, só get derivado.
final class Person {
    public function __construct(
        public string $first,
        public string $last,
    ) {}

    // Sintaxe curta: arrow style.
    public string $fullName {
        get => trim($this->first . ' ' . $this->last);
    }

    // Sintaxe longa: bloco com return.
    public string $initials {
        get {
            return strtoupper(($this->first[0] ?? '') . ($this->last[0] ?? ''));
        }
    }
}

$p = new Person('Ana', 'Silva');
echo $p->fullName;  // "Ana Silva"
echo $p->initials;  // "AS"
// $p->fullName = 'X';  // Error: cannot write to virtual property without set hook.
```

```php
<?php
declare(strict_types=1);

// 3) Hooks em interface: contrato de propriedade.
interface HasIdentity {
    public string $id { get; }                  // read-only via interface
    public string $name { get; set; }           // read/write
}

final class Account implements HasIdentity {
    public function __construct(
        public readonly string $id,             // readonly satisfaz "get"
        public string $name,                    // propriedade simples satisfaz get+set
    ) {}
}

// Implementação alternativa com hook backed.
final class LoggedAccount implements HasIdentity {
    public string $id { get => $this->_id; }
    public string $name {
        get => $this->_name;
        set { error_log("name changed to {$value}"); $this->_name = $value; }
    }

    public function __construct(private string $_id, private string $_name) {}
}
```

```php
<?php
declare(strict_types=1);

// 4) Hooks + herança: sobrescrita parcial.
class Base {
    public string $title {
        get => $this->title;
        set { $this->title = ucfirst($value); }
    }
}

final class Article extends Base {
    // Só sobrescreve o get; set permanece o do parent.
    public string $title {
        get => '[Article] ' . parent::$title::get();   // sintaxe ilustrativa
    }
}
// Atenção: na prática, sobrescrita parcial tem limitações;
// quando precisar de duas lógicas distintas, é mais limpo redeclarar ambos.
```

```php
<?php
declare(strict_types=1);

// 5) Cache de cálculo caro: hook + propriedade backing privada.
final class Report {
    private ?array $_cache = null;

    public function __construct(private readonly array $rows) {}

    public array $summary {
        get {
            // Cálculo caro: cache lazy.
            return $this->_cache ??= $this->computeSummary();
        }
    }

    private function computeSummary(): array {
        // Suponha O(n log n) sobre milhões de rows.
        return ['count' => count($this->rows), 'sum' => array_sum($this->rows)];
    }
}
$r = new Report([1, 2, 3, 4]);
echo $r->summary['sum'];  // calcula
echo $r->summary['sum'];  // retorna do cache
```

## Quando usar / quando não usar

- **Use `set` hook** quando há **um único invariante** ligado àquela propriedade (normalização, range, formato).
- **Use `get` virtual** para propriedades derivadas baratas (formatação, concatenação simples).
- **Use hooks em interface** quando o contrato deveria exigir propriedade, não método — APIs mais coesas.
- **Use com cache lazy** quando o `get` é caro mas idempotente dentro da vida do objeto.
- **Não use** para lógica de I/O dentro de hooks — leitura silenciosa de banco em `get` viola o princípio de menor surpresa.
- **Não use** se a classe deveria ser readonly: `set` hook contradiz imutabilidade; valide no construtor.
- **Não use** em hot path de loops massivos sem medir — overhead é real comparado a propriedade pura.
- **Não use** quando vários invariantes cruzados envolvem múltiplas propriedades — método de domínio é mais claro.

## Armadilhas comuns

- **Atribuir `$this->prop = $value` dentro do `set` não recursiona**: o motor diferencia "escrever no storage" de "passar pelo hook". Confunde quem espera o comportamento de getter/setter em outras linguagens onde recursão é problema real.
- **`get => ...` sem `set` cria propriedade efetivamente read-only externamente**: tentar escrever lança `Error`. Útil, mas pode pegar de surpresa caller que esperava `public` significa "tudo permitido".
- **Virtual property não aparece em `get_object_vars` da mesma forma**: a propriedade existe no tipo mas não no array de instância. Serializers ingenuos podem omitir o campo derivado — verifique o suporte.
- **Hook `set` com tipo diferente da declaração**: `public string $x { set(int $v) { ... } }` é proibido em strict — o tipo do parâmetro do hook precisa ser compatível com o tipo da propriedade. Erro confunde porque parece flexível, mas não é.
- **Performance em loops**: cada acesso vira call. Em código que processa milhões de objetos, mediu-se overhead de 2-5x vs propriedade pura. Acontece porque o motor não inlinea hooks ainda.
- **Interação com `readonly`**: `readonly` + `set` hook é proibido — não faz sentido escrever uma vez via hook arbitrário. `readonly` + `get` virtual é permitido e útil.
- **Sobrescrita parcial em herança é restrita**: você não pode sobrescrever só o `set` deixando o `get` herdado de forma transparente. Geralmente precisa redeclarar ambos. Acontece porque a propriedade vira "nova" no subtipo.
- **Não confunda com `__get`/`__set` mágicos**: hooks são por propriedade declarada e tipada; mágicos são fallback genérico para propriedades não declaradas. Hooks são estáticos, verificáveis, performam melhor; mágicos são dinâmicos e fogem de análise estática.

## Links relacionados

- [[Readonly-Classes]]
- [[Asymmetric-Visibility]]
- [[Arquitetura/Value-Objects]]
- [[Constructor-Promotion]]
- [[Sistema-de-Tipos]]
- [[Atributos]]

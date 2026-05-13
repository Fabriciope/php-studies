---
tags: [php, oop, atributos, reflection]
fase: 2
status: draft
---

# Atributos (Attributes)

## Conceito

Atributos, introduzidos no PHP 8.0 pela RFC "Attributes v2", são **metadados estruturados** anexáveis a elementos do código — classes, métodos, propriedades, parâmetros, constantes de classe e (a partir de 8.1) constantes globais e enum cases. A sintaxe é `#[NomeDoAtributo(args...)]`, escrita acima do elemento decorado. Diferente de docblocks `@Annotation` (Doctrine Annotations), atributos são **parte da linguagem**: parseados pelo PHP, validados em sintaxe e armazenados em metadata da Reflection sem parser externo.

A mecânica: você declara uma classe e marca com `#[Attribute]`. Outra parte do código usa essa classe como decorador via `#[MinhaClasse(arg1, arg2)]`. Em runtime, `Reflection*::getAttributes()` retorna um array de `ReflectionAttribute`, cada um descrevendo a aplicação encontrada. Os argumentos não são executados na declaração — ficam guardados como literais. Só ao chamar `$ra->newInstance()` o PHP instancia a classe, valida tipos dos argumentos contra o construtor, e devolve o objeto utilizável.

Atributos aceitam configuração de **alvo** (`Attribute::TARGET_CLASS`, `TARGET_METHOD`, `TARGET_PROPERTY`, `TARGET_PARAMETER`, `TARGET_CLASS_CONSTANT`, `TARGET_FUNCTION`, `TARGET_ALL`) combináveis com `|`. Flags adicionais: `Attribute::IS_REPEATABLE` (permite o mesmo atributo aparecer múltiplas vezes no mesmo alvo) e `Attribute::IS_INHERITED` (que em PHP, na prática, não existe como flag nativa — atributos são lidos por padrão na classe alvo; herança é responsabilidade de quem chama `getAttributes`, que aceita flag `ReflectionAttribute::IS_INSTANCEOF`).

Diferença para docblock annotations (Doctrine): docblocks eram strings que precisavam parser regex/lexer externo; sofriam com strip-comments de opcaches, com erros de digitação que ficavam invisíveis, com falta de IDE support. Atributos eliminam isso: o IDE conhece a classe, o refactor renomeia, o analisador estático lê tipos.

Casos de uso típicos: **rotas** (Symfony `#[Route('/path')]`), **validação** (`#[Assert\NotBlank]`), **mapping ORM** (`#[ORM\Column]`), **DI** (`#[Inject]`, `#[Autowire]`), **serialização** (`#[Groups(['public'])]`), **comandos de console** (`#[AsCommand('app:foo')]`). Em todos esses casos, a configuração é **estática** — pertence ao tipo, não muda em runtime. Esse é o critério essencial.

Performance: ler atributos via reflection é relativamente caro (parser de metadata + alocação). Soluções modernas (Symfony, Laravel) cacheiam resultados de leitura em produção via container compilado ou cache de mapping. Em desenvolvimento, leitura por request; em produção, dump uma vez.

## Por que importa

**Cenário 1 — declaração junto do código que importa**: regras de validação ficam na classe DTO, não num XML separado. Quando o programador edita a classe, vê e atualiza as regras. Reduz drift entre código e configuração.

**Cenário 2 — frameworks plug-and-play**: você anota `#[Route('/users/{id}', methods: ['GET'])]` num método. O framework descobre via reflection sem você registrar a rota manualmente em outro arquivo. O custo de adicionar/remover rota cai para zero.

**Cenário 3 — refatoração segura**: renomear `Email` para `EmailAddress` propaga via IDE inclusive nos atributos (`#[Assert\Type(EmailAddress::class)]`). Com docblock string `@Assert\Type("Email")` o refactor falhava silenciosamente.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// 1) Declarar um atributo simples.
#[Attribute(Attribute::TARGET_CLASS)]
final readonly class Route {
    public function __construct(
        public string $path,
        public array $methods = ['GET'],
    ) {}
}

// Uso:
#[Route('/users', methods: ['GET', 'POST'])]
final class UsersController {
    public function index(): array { return []; }
}

// Leitura via reflection.
$ref = new ReflectionClass(UsersController::class);
foreach ($ref->getAttributes(Route::class) as $attr) {
    $route = $attr->newInstance();   // instancia DE VERDADE aqui.
    echo $route->path;               // '/users'
    print_r($route->methods);        // ['GET', 'POST']
}
```

```php
<?php
declare(strict_types=1);

// 2) Atributo repetível em propriedades — validação declarativa.
#[Attribute(Attribute::TARGET_PROPERTY | Attribute::IS_REPEATABLE)]
final readonly class Validate {
    public function __construct(
        public string $rule,
        public mixed $arg = null,
    ) {}
}

final class CreateUser {
    #[Validate('required')]
    #[Validate('max', 120)]
    public string $name;

    #[Validate('email')]
    #[Validate('required')]
    public string $email;
}

// Reader genérico.
/** @return array<string, list<Validate>> */
function rulesOf(string $class): array {
    $rules = [];
    $ref = new ReflectionClass($class);
    foreach ($ref->getProperties() as $prop) {
        foreach ($prop->getAttributes(Validate::class) as $attr) {
            $rules[$prop->getName()][] = $attr->newInstance();
        }
    }
    return $rules;
}

$rules = rulesOf(CreateUser::class);
// [
//   'name'  => [Validate('required'), Validate('max', 120)],
//   'email' => [Validate('email'),    Validate('required')],
// ]
```

```php
<?php
declare(strict_types=1);

// 3) Alvo múltiplo + ler atributos com filtro de instanceof.
interface DocumentationTag {}

#[Attribute(Attribute::TARGET_CLASS | Attribute::TARGET_METHOD)]
final readonly class Deprecated implements DocumentationTag {
    public function __construct(public string $reason, public string $since) {}
}

#[Attribute(Attribute::TARGET_CLASS)]
final readonly class Experimental implements DocumentationTag {
    public function __construct(public string $reason) {}
}

#[Deprecated(reason: 'use NewService', since: '2.0')]
#[Experimental(reason: 'API may change')]
final class OldService {
    #[Deprecated(reason: 'use doNew()', since: '2.1')]
    public function doOld(): void {}
}

$ref = new ReflectionClass(OldService::class);
// IS_INSTANCEOF: pega qualquer atributo que implemente DocumentationTag.
foreach ($ref->getAttributes(DocumentationTag::class, ReflectionAttribute::IS_INSTANCEOF) as $a) {
    $tag = $a->newInstance();
    echo get_class($tag) . ': ' . $tag->reason . "\n";
}
```

```php
<?php
declare(strict_types=1);

// 4) Validação dos argumentos do atributo no newInstance().
#[Attribute(Attribute::TARGET_PROPERTY)]
final readonly class Range {
    public function __construct(public int $min, public int $max) {
        if ($min > $max) {
            throw new InvalidArgumentException("min ($min) > max ($max)");
        }
    }
}

final class Score {
    #[Range(min: 10, max: 0)]   // ERRO só DISPARA no newInstance().
    public int $value;
}

// Parsing do atributo NÃO valida — só na hora de instanciar.
$attr = (new ReflectionProperty(Score::class, 'value'))->getAttributes()[0];
try {
    $attr->newInstance();
} catch (\Throwable $e) {
    echo $e->getMessage();   // "min (10) > max (0)"
}
// Lição: a verificação real dos argumentos é responsabilidade do construtor.
```

```php
<?php
declare(strict_types=1);

// 5) Cache de leitura: padrão de produção.
final class AttributeRegistry {
    /** @var array<class-string, array<string, list<object>>> */
    private static array $cache = [];

    /** @return list<object> */
    public static function read(string $class, string $element, string $attrClass): array {
        $key = "$class::$element:$attrClass";
        if (isset(self::$cache[$key])) {
            return self::$cache[$key];
        }
        $ref = new ReflectionClass($class);
        $target = $element === '' ? $ref : $ref->getProperty($element);
        $list = [];
        foreach ($target->getAttributes($attrClass) as $a) {
            $list[] = $a->newInstance();
        }
        return self::$cache[$key] = $list;
    }
}
// Em produção, prefira container compilado (Symfony Container Builder,
// Laravel container cache) que faz o dump em build time — zero reflection
// no hot path.
```

## Quando usar / quando não usar

- **Use** para configuração **declarativa** que pertence ao tipo: rotas, validação, mapping ORM, hints de DI, serialização, comandos.
- **Use** quando o consumidor é um framework/library que lê via reflection e cacheia em produção.
- **Use** atributos repetíveis (`IS_REPEATABLE`) para regras múltiplas no mesmo alvo (várias validações por campo).
- **Use** hierarquia + `IS_INSTANCEOF` para agrupar atributos por interface marker (`DocumentationTag`, `RouteAttribute`).
- **Não use** para configuração que muda em runtime (feature flags, A/B testing) — atributos são estáticos por design.
- **Não use** como substituto de método: se a lógica do "atributo" depende de estado de instância, é método de domínio, não metadado.
- **Não use** se você não tem reader: atributo sem ninguém lendo é só ruído visual.
- **Não abuse**: 6 atributos numa classe simples sugere que falta abstração ou que a classe carrega responsabilidades demais.

## Armadilhas comuns

- **Esquecer `#[Attribute]` na classe alvo**: a classe vira atributo "fantasma". Reflection retorna o `ReflectionAttribute`, mas `newInstance()` lança `Error: Attribute class is not marked with #[Attribute]`. Acontece porque sem a marcação o PHP recusa instanciar.
- **`IS_REPEATABLE` ausente em atributo repetido**: usar `#[Foo] #[Foo]` sem `IS_REPEATABLE` é erro de compilação. Acontece porque o PHP precisa saber em parse-time se a duplicação é intencional.
- **Argumentos do atributo são literais — sem `new` nem chamadas**: `#[Foo(new Bar())]` falha. Só constantes, enums, arrays de literais. Acontece porque atributos são metadata estática; runtime acontece só no `newInstance`.
- **Performance de reflection sem cache**: ler atributos em cada request mata performance. Acontece porque `getAttributes` re-aloca; produção exige cache (preload, container compilado).
- **Atributos não são herdados automaticamente**: `class Child extends Parent` não recebe atributos da parent ao chamar `getAttributes()` na child. Acontece porque PHP não implementa herança de atributo nativamente — quem precisa, percorre a cadeia manualmente.
- **Validação só acontece em `newInstance()`**: atributos com argumentos inválidos não falham no parse. Acontece porque o construtor só roda no instance — daí a importância de validar no `__construct` do atributo. Bugs ficam latentes até o reader rodar.
- **Atributos em parâmetros promovidos são duplicados**: `#[Foo] private string $x` em promotion aparece tanto em `ReflectionParameter::getAttributes()` quanto em `ReflectionProperty::getAttributes()`. Acontece porque promoted vira parâmetro E propriedade. Decida em qual nível ler para não processar duas vezes.
- **`Attribute::TARGET_ALL` esconde erros de uso**: usar `TARGET_ALL` permite que o atributo apareça em qualquer lugar — inclusive em locais sem sentido. Acontece porque o PHP só valida o alvo se o flag for restritivo. Prefira alvos específicos para que erros de uso sejam compilation errors.

## Links relacionados

- [[Readonly-Classes]]
- [[Sistema-de-Tipos]]
- [[Padroes/SOLID]]
- [[Constructor-Promotion]]
- [[Enums]]
- [[Property-Hooks]]

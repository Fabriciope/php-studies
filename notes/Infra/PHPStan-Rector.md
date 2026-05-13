---
tags: [php, infra, qualidade, phpstan, rector]
fase: 7
status: draft
---

# PHPStan + Rector

## Conceito

**PHPStan** é um analisador estático para PHP: lê o código sem executá-lo, constrói um modelo de tipos e fluxo, e reporta inconsistências — métodos chamados em `null`, propriedades não inicializadas, retornos que mentem sobre o tipo, branches mortos, variáveis nunca usadas. A análise estática não substitui testes (não valida regra de negócio), mas elimina uma classe inteira de bugs que testes raramente cobrem: erros de tipo em caminhos pouco exercitados.

O modelo mental: PHPStan **simula a execução** do programa com todos os possíveis valores que cada expressão poderia assumir, segundo os tipos declarados (em sintaxe PHP ou em PHPDoc). Quando essa simulação encontra uma operação impossível (ex.: chamar `->name` num valor que poderia ser `null`), reporta. Quanto mais alto o **nível** configurado (0 a 9, mais `max`), mais rigorosa é a simulação.

| Nível | O que verifica adicionalmente |
|------:|---|
| 0 | Classes/métodos/funções básicos existem |
| 1 | Variáveis e métodos não definidos em closures |
| 2 | Métodos desconhecidos em qualquer expressão tipada |
| 3 | Tipos de retorno e atribuições inconsistentes |
| 4 | Dead code (branches inalcançáveis) |
| 5 | Tipos de argumentos passados em chamadas |
| 6 | PHPDoc faltando (parâmetros e retornos) |
| 7 | Tipos parciais (union mal estreitada) |
| 8 | **`null` vs not-null** — checagem agressiva |
| 9 | `mixed` proibido como tipo "real" |
| max | Tudo de 9 + futuras checagens |

Em produção séria, o alvo é **nível 8** com `treatPhpDocTypesAsCertain: false`. Nível 9 é desejável para bibliotecas; pode ser excessivo em aplicações com integrações legadas.

**Rector** é um motor de **refactoring automático**: aplica transformações AST baseadas em **regras**. As regras vêm agrupadas em **sets** — coleções temáticas: `LevelSetList::UP_TO_PHP_84` (moderniza sintaxe para PHP 8.4), `SetList::CODE_QUALITY` (early returns, simplificações), `SetList::DEAD_CODE` (remove código morto), `SetList::TYPE_DECLARATION` (adiciona tipos em parâmetros e retornos baseado em inferência), `SetList::PRIVATIZATION` (visibilidade mais restrita quando possível).

A diferença filosófica importa: **PHPStan reporta, Rector reescreve**. PHPStan é defesa (não deixa entrar); Rector é ofensa (transforma o que existe). Os dois se complementam — Rector pode gerar código que ainda precisa de revisão por PHPStan para garantir que nada quebrou.

PHPDocs estendidos viram parte do **sistema de tipos** efetivo: `@template` para generics, `@phpstan-impl`, `@phpstan-assert`, `@phpstan-pure`, `@psalm-immutable`, `@phpstan-type` (alias de tipos), conditional return types (`@return ($flag is true ? string : null)`). Esses anotações sobem o teto da expressividade muito além do que o engine PHP entende em runtime, dando à análise estática poder próximo de TypeScript.

## Por que importa

Análise estática rigorosa **muda como o código é escrito**. Em projeto novo com PHPStan 8 desde o dia zero, o desenvolvedor adquire reflexo de tipar honestamente — retornos `?User` em vez de `User|false|null`, value objects ao invés de arrays primitivos, enums ao invés de strings mágicas. Bugs por `null` reference, off-by-one em arrays, branches mortos, esses simplesmente não chegam ao review.

Em **legacy**, o ganho aparece em duas frentes: primeiro como mapa de minas (o baseline mostra onde estão os tipos suspeitos), depois como rede de proteção (subir um nível por sprint estabelece progresso mensurável).

**Rector ganha valor em três cenários**:

1. **Upgrade de versão PHP**: subir de 8.2 → 8.4 com `LevelSetList::UP_TO_PHP_84` aplica em segundos o que seria semana de trabalho manual: readonly properties onde cabem, named arguments, first-class callable syntax, enum onde havia constantes.
2. **Modernização contínua**: rodar `SetList::CODE_QUALITY` mensalmente mantém código idiomático sem virar projeto separado.
3. **Refactor em massa**: renomear método em 200 classes, migrar de uma lib para outra, adicionar tipo de retorno em todo lugar — tudo via regra customizada.

**Cenário concreto**: equipe assume codebase com 8 anos de PHP 5.6/7.x. Sem Rector, modernização é projeto de meses. Com Rector + PHPStan, a primeira semana já entrega: `array_filter` com `ARRAY_FILTER_USE_BOTH` documentado, `null coalescing` em vez de ternários verbosos, type hints inferidos em parâmetros, dead code removido, readonly em DTOs. PHPStan então valida que nada quebrou conceitualmente.

## Exemplo de código PHP

```neon
# phpstan.neon — configuração de referência
includes:
    - vendor/phpstan/phpstan-strict-rules/rules.neon
    - vendor/phpstan/phpstan-deprecation-rules/rules.neon
    - vendor/phpstan/phpstan-phpunit/extension.neon
    - phpstan-baseline.neon

parameters:
    level: 8
    paths:
        - src
        - tests
    excludePaths:
        analyse:
            - src/Generated/*
            - src/Legacy/Vendor/*

    # comportamento da análise
    treatPhpDocTypesAsCertain: false       # NUNCA confie cegamente em PHPDoc
    checkUninitializedProperties: true     # propriedades typed precisam init
    checkBenevolentUnionTypes: true        # union types validados estrito
    checkImplicitMixed: true               # mixed implícito vira erro
    reportUnmatchedIgnoredErrors: true     # ignores obsoletos viram erro

    # caching
    tmpDir: .phpstan-cache

    # ignores devem ser raríssimos e com justificativa
    ignoreErrors:
        -
            message: '#Call to an undefined method DateTimeInterface::add\(\)#'
            path: src/Infrastructure/LegacyClock.php
            count: 1
            # TODO: remover quando #ISSUE-1234 resolver

    # types globais reutilizáveis
    typeAliases:
        UserId: 'string'
        Money: 'array{amount: int, currency: string}'
```

```php
<?php
// rector.php
declare(strict_types=1);

use Rector\Config\RectorConfig;
use Rector\Set\ValueObject\LevelSetList;
use Rector\Set\ValueObject\SetList;
use Rector\CodingStyle\Rector\Encapsed\EncapsedStringsToSprintfRector;

return RectorConfig::configure()
    ->withPaths([
        __DIR__ . '/src',
        __DIR__ . '/tests',
    ])
    ->withSkip([
        // arquivos onde Rector não toca
        __DIR__ . '/src/Generated',
        EncapsedStringsToSprintfRector::class,    // regra que rejeitamos
    ])

    // sobe a sintaxe gradativamente até PHP 8.4
    ->withPhpSets(php84: true)

    // sets temáticos
    ->withSets([
        SetList::CODE_QUALITY,        // early return, simplificações
        SetList::DEAD_CODE,           // remove código morto
        SetList::TYPE_DECLARATION,    // adiciona tipos onde inferível
        SetList::EARLY_RETURN,        // achata aninhamentos
        SetList::PRIVATIZATION,       // visibilidade mais restrita
        SetList::NAMING,              // renomeia variáveis confusas
    ])

    // remove use statements órfãos
    ->withImportNames(
        importNames: true,
        importDocBlockNames: true,
        importShortClasses: false,
        removeUnusedImports: true,
    )

    // cache para CI rápido
    ->withCache(__DIR__ . '/.rector-cache');
```

```php
<?php
// Exemplos de PHPDoc avançado que PHPStan entende

/**
 * Generic container. T é parâmetro de tipo.
 *
 * @template T of object
 */
final class Repository
{
    /** @var class-string<T> */
    private string $entityClass;

    /**
     * @param class-string<T> $entityClass
     */
    public function __construct(string $entityClass) {
        $this->entityClass = $entityClass;
    }

    /** @return T|null */
    public function find(string $id): ?object { /* ... */ }

    /** @return list<T> */
    public function all(): array { /* ... */ }
}

// uso: PHPStan infere o tipo concreto
$repo = new Repository(User::class);   // Repository<User>
$user = $repo->find('123');             // User|null
$user?->email();                        // OK

/**
 * Conditional return type: retorno depende do valor de $strict.
 *
 * @return ($strict is true ? User : User|null)
 */
function load(string $id, bool $strict = false): ?User { /* ... */ }

/**
 * Assertion: depois desta função, $value é garantido int.
 *
 * @phpstan-assert int $value
 */
function ensureInt(mixed $value): void {
    if (!is_int($value)) {
        throw new InvalidArgumentException();
    }
}

/**
 * Pure: sem side effects. PHPStan otimiza assumindo isso.
 *
 * @phpstan-pure
 */
function add(int $a, int $b): int { return $a + $b; }
```

```php
<?php
// Regra Rector customizada — exemplo simples
namespace App\Rector;

use PhpParser\Node;
use PhpParser\Node\Expr\MethodCall;
use Rector\Rector\AbstractRector;
use Rector\RectorDefinition\CodeSample;
use Rector\RectorDefinition\RectorDefinition;

/**
 * Substitui ->save() seguido de ->refresh() por ->saveAndRefresh().
 */
final class UnifySaveRefreshRector extends AbstractRector
{
    public function getNodeTypes(): array
    {
        return [MethodCall::class];
    }

    public function refactor(Node $node): ?Node
    {
        if (!$node instanceof MethodCall) return null;
        if (!$this->isName($node->name, 'refresh')) return null;

        $prev = $node->var;
        if (!$prev instanceof MethodCall) return null;
        if (!$this->isName($prev->name, 'save')) return null;

        return new MethodCall(
            $prev->var,
            'saveAndRefresh',
            $prev->args,
        );
    }
}
```

## Estratégia de adoção em legacy

Roteiro testado para subir um codebase legado a PHPStan nível 8 sem parar features:

1. **Configurar nível 0 + gerar baseline**: `vendor/bin/phpstan analyse --generate-baseline`. O baseline congela o estado atual. Daí em diante, **novos erros falham o CI**, mas os antigos não bloqueiam.
2. **Subir um nível por sprint**, regenerando baseline. Cada subida desentupes uma classe de bug.
3. **Baseline com dono e prazo**: cada entrada no baseline ganha comentário com `TODO`, owner e data limite. Sem isso, baseline vira lixão eterno.
4. **CI guard contra crescimento do baseline**: PR que adicione entrada ao baseline reprovado (ver workflow `phpstan-baseline-guard.yml` em [[CI-CD]]).
5. **Rector aplicado por set, em PR isolado**: nunca misture `code_quality` com `php_84` no mesmo PR. Review fica impraticável. Um set por PR, com checklist de validação.
6. **Estabelecer code review checklist**: novo código precisa passar PHPStan no nível alvo **sem entrar no baseline**.
7. **Mensalmente, reduzir baseline em N%**. Cria pressão saudável.

Tabela de critério para decidir entre baseline vs corrigir agora:

| Situação | Decisão |
|---|---|
| Erro em código tocado pelo PR | Corrigir agora |
| Erro em módulo crítico de negócio | Sprint específica |
| Erro em código de boundary com lib externa fora do controle | Baseline + ignoreErrors documentado |
| Erro em código gerado | `excludePaths` |
| Erro em teste antigo | Baseline com prazo curto |

## Quando usar / quando não usar

- **PHPStan nível 8 sem baseline** — alvo para qualquer código novo de aplicação.
- **PHPStan max** — alvo para bibliotecas distribuídas via Packagist.
- **Strict rules + deprecation rules** — adicione sempre, custo zero.
- **Psalm como segunda opinião** — opcional. Para a maior parte dos times, escolher um (PHPStan ou Psalm) e usar bem é melhor que rodar os dois mal.
- **Rector em CI `--dry-run` em todo PR** — sempre. Aplicação efetiva sempre em PR humano revisado.
- **Não use** Rector em modo automático no main sem PR de revisão. Mudança em massa precisa de humano olhando.
- **Não use** `treatPhpDocTypesAsCertain: true` — esse foi default no PHPStan antigo e esconde bugs reais.
- **Não use** PHPStan apenas em diretório `tests` — análise mais útil é em `src`.

## Armadilhas comuns

- **`treatPhpDocTypesAsCertain: true`**: confia cegamente em PHPDoc, esconde bugs onde a anotação mente sobre a realidade. Bug clássico: classe anota `@return User` mas em algum caminho retorna `null` — com `true`, PHPStan acredita; com `false`, denuncia.
- **Baseline como lista permanente**: sem prazo nem owner, baseline vira lixão e o nível alto deixa de proteger. Crie processo de redução.
- **Rector aplicado em PR gigante (10k linhas, vários sets misturados)**: review impossível, bugs entram disfarçados de "refactor". Um set por PR, sempre.
- **`@phpstan-ignore` sem comentário do porquê**: rastro perdido. Daqui a 6 meses ninguém sabe se ainda é válido. Sempre comente.
- **PHPStan rodando sem cache em CI**: análise leva 3 minutos a mais por job. Sempre cacheie `tmpDir`.
- **Confiar em Rector cegamente para mudanças semânticas**: regras de `CODE_QUALITY` às vezes alteram comportamento sutil (ex.: short-circuit em condições). Sempre revise diff.
- **Não atualizar Rector e PHPStan**: regras novas trazem checagens importantes (ex.: PHP 8.4 deprecations). Mantenha atualizados via Renovate/Dependabot.
- **Configurar `level: max` no dia zero em legacy**: 5000 erros, time desiste, PHPStan vira "aquela coisa que ignora". Comece em 0 e suba.
- **Excluir `tests` da análise**: testes têm bugs também, e PHPStan ajuda a refatorar fixtures. Inclua `tests`, talvez com nível 1 abaixo.

## Links relacionados

- [[CI-CD]]
- [[Fundamentos/Strict-Types]]
- [[OOP-Moderno/Atributos]]
- [[OOP-Moderno/Readonly-Enums-Match]]
- [[Tests/Pest-PHPUnit]]
- [[Fundamentos/Composer]]

---
tags: [php, infra, qualidade, phpstan, rector]
fase: 7
status: stub
---

# PHPStan + Rector

## Conceito

- **PHPStan** — análise estática. Encontra bugs de tipo, métodos inexistentes, dead code, variáveis não usadas. **Não** roda código. 10 níveis (0–9 + level max). Em produção sério, **nível 8** sem baseline é o alvo.
- **Rector** — refactor automático. Aplica regras (sets) para modernizar código (`code_quality`, `dead_code`, `php_84`, `type_declaration`). Idempotente; roda no CI em modo `--dry-run` para detectar drift.

## Por que importa

Análise estática rigorosa **muda como o código é escrito** — força tipos honestos, elimina classes inteiras de bugs antes do teste rodar. Rector mantém a base atualizada sem maratona manual: subir de PHP 8.2 para 8.4 vira `composer rector` em vez de sprint.

## Exemplo de código

```neon
# phpstan.neon
includes:
    - vendor/phpstan/phpstan-strict-rules/rules.neon

parameters:
    level: 8
    paths:
        - src
        - tests
    excludePaths:
        - src/Generated/**
    treatPhpDocTypesAsCertain: false
    checkUninitializedProperties: true
    checkBenevolentUnionTypes: true
    reportUnmatchedIgnoredErrors: true
    # baseline temporário com data de validade
    # baseline: phpstan-baseline.neon
```

```php
<?php
// rector.php
declare(strict_types=1);

use Rector\Config\RectorConfig;
use Rector\Set\ValueObject\LevelSetList;
use Rector\Set\ValueObject\SetList;

return RectorConfig::configure()
    ->withPaths([__DIR__ . '/src', __DIR__ . '/tests'])
    ->withPhpSets(php84: true)
    ->withSets([
        SetList::CODE_QUALITY,
        SetList::DEAD_CODE,
        SetList::TYPE_DECLARATION,
        SetList::EARLY_RETURN,
    ])
    ->withImportNames(removeUnusedImports: true);
```

## Estratégia de adoção em legacy

1. **Subir nível gradualmente**: começar em 0/1, gerar baseline, subir um nível por sprint.
2. **Baseline com prazo**: cada erro do baseline tem dono e deadline (max 60 dias).
3. **Rector em PR único por set**: não misturar `code_quality` com `php_84` — fica impossível revisar.
4. **Bloqueio de regressão**: PR que aumenta erros do baseline reprova.

## Quando usar / quando não usar

- **PHPStan nível 8** — sempre em código novo. Em legacy, plano de chegada.
- **Psalm como segunda opinião** — opcional. PHPStan + Rector cobrem a maior parte.
- **Rector em CI `--dry-run`** sempre. Aplicação em PRs separados.
- **Não usar** Rector "automaticamente" em main sem revisão humana — mudança em massa precisa de PR e revisão.

## Armadilhas comuns

- `treatPhpDocTypesAsCertain: true` (default antigo) — confia em PHPDoc, esconde bugs.
- Baseline tratado como "lista permanente" — vira lixo. Use prazo.
- Rector aplicado num único PR gigante (10k linhas) — review impossível.
- Ignorar erro com `// @phpstan-ignore` sem comentário do porquê — rastro perdido.

## Links relacionados

- [[CI-CD]]
- [[Fundamentos/Strict-Types]]
- [[OOP-Moderno/Atributos]]

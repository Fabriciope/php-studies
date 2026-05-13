---
tags: [php, composer, dependencias, autoload, fundamentos]
fase: 1
status: draft
---

# Composer — gerenciador de dependências

## Conceito

**Composer** é o gerenciador de dependências oficial do ecossistema PHP. Inspirado em npm (Node) e bundler (Ruby), nasceu em 2012 (Nils Adermann e Jordi Boggiano) e mudou a face da linguagem: antes, copiava-se libs manualmente para `lib/`; depois, declara-se em `composer.json` e ele resolve, baixa e organiza tudo. Foi o que permitiu PHP ter um ecossistema **interoperável** via PSRs (PSR-4 autoload é Composer-driven).

Tecnicamente, Composer é uma **CLI escrita em PHP** que faz três coisas:

1. **Resolução de dependências** — dado um grafo de "este projeto precisa de A, A precisa de B^2.0, etc", encontra um conjunto de versões compatível com todas as constraints (SAT solver).
2. **Download** — busca cada pacote em [Packagist](https://packagist.org) (registry público) ou repositórios alternativos (privados, VCS, path).
3. **Autoload** — gera `vendor/autoload.php` que faz lazy-loading de classes via convenção PSR-4 (`Vendor\Namespace\Class` ↔ `src/Class.php`).

Dois arquivos centrais:

| Arquivo            | Papel                                                              | Versionado? |
| ------------------ | ------------------------------------------------------------------ | ----------- |
| `composer.json`    | Declarativo: constraints, autoload, scripts, metadata              | Sim         |
| `composer.lock`    | Snapshot exato das versões resolvidas (commit hash, hash do zip)   | Sim         |

A regra de ouro: **commite o lock**. Sem ele, `composer install` em outra máquina pode resolver versões diferentes — bug em produção que ninguém reproduz local.

**Versionamento semântico (SemVer)** rege as constraints: `MAJOR.MINOR.PATCH` onde major quebra contrato, minor adiciona feature sem quebrar, patch corrige bug. Composer usa essa convenção para interpretar operadores:

- `^1.2.3` → `>=1.2.3 <2.0.0` (tilde safe — caret default)
- `~1.2.3` → `>=1.2.3 <1.3.0` (mais restritivo)
- `1.2.*` → `>=1.2.0 <1.3.0` (wildcard)
- `>=1.2.3` → tudo daí pra cima (perigoso, evite)
- `1.2.3` → exatamente essa (raro; só em legados problemáticos)

## Por que importa

Composer **não é opcional** em PHP profissional. Todo framework, lib séria, ferramenta de qualidade (PHPStan, PHPUnit, Rector) é distribuída como pacote Composer. Sem ele, você está fora do ecossistema.

**Cenário 1 — onboarding de devs.** Sem Composer: "instale o framework X manualmente, copie estas libs do FTP, ajuste estes paths no `require_once`". Com Composer: `git clone && composer install`. Diferença entre 4 horas e 4 minutos.

**Cenário 2 — atualização de segurança.** Vulnerabilidade crítica em `guzzlehttp/guzzle 7.3`. Sem Composer: caça manual de onde está cada cópia, replace, teste, reza. Com Composer: `composer update guzzlehttp/guzzle` + CI verde, deploy.

**Cenário 3 — autoload performance.** Em produção, `composer dump-autoload --optimize --classmap-authoritative` gera um classmap estático: tabela hash de classes → arquivos. Resolve `new Foo()` em O(1) sem stat() em disco. Em apps grandes, ganho é mensurável (10-30% no boot).

**Cenário 4 — supply chain attack.** Pacote popular comprometido por mantenedor mal-intencionado. `composer audit` (built-in desde 2.4) checa CVEs locais contra base GitHub Security Advisories — alerta antes de você merge-ar uma versão venenosa.

## Exemplo de código PHP

```bash
# === instalação (uma vez por máquina) ===
# macOS / Linux
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer

# verificar
composer --version
# Composer version 2.7.x

# atualizar o próprio Composer (raramente)
composer self-update
```

```bash
# === bootstrap de projeto novo ===
mkdir meu-app && cd meu-app
composer init                       # wizard interativo

# adicionar dependência de runtime
composer require monolog/monolog
composer require symfony/http-foundation "^7.0"

# adicionar dev-only (PHPUnit, PHPStan, Rector)
composer require --dev phpunit/phpunit "^11.0"
composer require --dev phpstan/phpstan "^1.11"

# instalar tudo (em outra máquina, após clone)
composer install                    # lê lock; reprodutível
composer install --no-dev           # produção: sem dev deps

# atualizar (mexe no lock)
composer update                     # tudo respeitando constraints
composer update vendor/pacote       # apenas este

# auditoria de vulnerabilidades
composer audit
```

```json
// composer.json típico de projeto sério
{
    "name": "fabricio/orders-api",
    "description": "API REST de pedidos",
    "type": "project",
    "license": "proprietary",
    "require": {
        "php": "^8.3",
        "ext-pdo": "*",
        "ext-redis": "*",
        "monolog/monolog": "^3.5",
        "symfony/http-foundation": "^7.0",
        "ramsey/uuid": "^4.7"
    },
    "require-dev": {
        "phpunit/phpunit": "^11.0",
        "phpstan/phpstan": "^1.11",
        "rector/rector": "^1.0"
    },
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
    },
    "scripts": {
        "test": "phpunit",
        "stan": "phpstan analyse --memory-limit=512M",
        "rector": "rector process --dry-run",
        "check": [
            "@stan",
            "@test"
        ]
    },
    "config": {
        "sort-packages": true,
        "optimize-autoloader": true,
        "platform": {
            "php": "8.3.0"
        }
    },
    "minimum-stability": "stable",
    "prefer-stable": true
}
```

```php
<?php
declare(strict_types=1);

// === uso: uma linha resolve tudo ===
require __DIR__ . '/vendor/autoload.php';

use Monolog\Logger;
use Monolog\Handler\StreamHandler;
use Ramsey\Uuid\Uuid;

$logger = new Logger('app');
$logger->pushHandler(new StreamHandler('php://stdout'));

$logger->info('request received', [
    'request_id' => Uuid::uuid7()->toString(),
]);
```

```bash
# === build de produção ===
composer install \
    --no-dev \
    --no-progress \
    --optimize-autoloader \
    --classmap-authoritative \
    --prefer-dist
```

> [!tip] Performance de autoload
> `--classmap-authoritative` substitui PSR-4 dinâmico por classmap estático. Resolve `new App\Foo()` em hash lookup; se a classe não está no mapa, **falha imediatamente** (não tenta filesystem). Em produção, ganho real e elimina race conditions de I/O.

## Quando usar / quando não usar

- **Sempre** em projeto novo ou legado migrável. Sem exceção.
- **`composer require`** para adicionar; nunca edite `composer.json` à mão (pode até, mas use a CLI por consistência de formatação).
- **`composer update` na hora certa**: PRs dedicados, leia changelog, rode testes. Não confunda com `install`.
- **Lock SEMPRE versionado.** Mesmo em libs (Symfony recomenda commit; Composer 2.x default agora é commitar).
- **`require-dev` para tudo que não roda em produção**: PHPUnit, PHPStan, Rector, faker.
- **Não use Composer** apenas para "evitar git submodules" se a dependência é seu próprio código não publicado. Use `repositories: path` para monorepo, ou publique em registry privado.

## Armadilhas comuns

- **Esquecer de commitar `composer.lock`.** Outra máquina/CI resolve versões diferentes, produzindo "funciona aqui, quebra lá". Lock é parte da fonte da verdade.
- **`composer update` em produção.** NUNCA. Update gera novo lock; deploy deve usar `install` (e idealmente o lock veio do CI). Update é tarefa de dev/CI dedicado.
- **Constraints frouxas (`>=1.0`).** Quando upstream lança 2.0 incompatível, seu próximo `install` resolve para 2.0 e quebra silenciosamente. Use `^` para SemVer-safe.
- **`composer install` sem `--no-dev` em produção.** Vai junto Xdebug profiler, PHPUnit, faker — bloat de imagem, surface de ataque maior, ferramentas que nunca deveriam tocar prod.
- **Ignorar `composer audit`.** Pacote vulnerável fica em produção meses até alguém escanear. Rode no CI; falhe a build se houver advisory de severidade alta/crítica.
- **Autoload sem `--optimize` em produção.** Cada `new` faz `stat()` no filesystem percorrendo prefixes PSR-4. Em apps grandes, 30-40% do bootstrap. Sempre otimize no build.
- **Pacotes "fantasma".** Sua app usa `Foo\Bar` que vem como dependência transitiva de `pkg/X`. Funciona até `pkg/X` remover a dep e seu app quebrar. Declare explicitamente o que importa.
- **Misturar `composer install` e `composer update` em CI.** Build reprodutível usa `install --no-dev`. Update fica em job manual ou agendado (renovate / dependabot).
- **Editar `vendor/`.** Toda alteração some no próximo `install/update`. Para patch em lib third-party: fork + `repositories: vcs`, ou `cweagans/composer-patches`.
- **`minimum-stability: dev` sem necessidade.** Aceita qualquer commit recente do branch de dev — produção pode pegar código instável. Mantenha `stable` por default; abra exceção pontual com `@dev` no pacote específico.
- **Conflict de versão de PHP entre dev e prod.** `composer.json` deve ter `"config.platform.php": "8.3.0"` setado para a versão de produção, mesmo se a máquina dev tem 8.4 — assim Composer resolve para libs compatíveis com 8.3.

## Comandos cheatsheet

```bash
composer init                        # wizard novo projeto
composer require vendor/pkg          # adicionar runtime dep
composer require --dev vendor/pkg    # adicionar dev dep
composer remove vendor/pkg           # remover
composer install                     # reproduzir lock
composer install --no-dev            # produção
composer update                      # atualizar dentro de constraints
composer update vendor/pkg           # apenas este
composer outdated                    # mostrar quais têm versão nova
composer audit                       # CVEs
composer show vendor/pkg             # info do pacote
composer why vendor/pkg              # quem depende dele
composer why-not vendor/pkg 2.0      # por que não dá pra subir
composer dump-autoload -o            # regerar autoload otimizado
composer validate                    # validar composer.json
composer diagnose                    # checagem geral
composer global require ...          # ferramenta global (psalm, etc)
```

## Composer × outros gerenciadores

| Aspecto             | Composer    | npm         | bundler     | pip         |
| ------------------- | ----------- | ----------- | ----------- | ----------- |
| Manifest            | composer.json | package.json | Gemfile  | requirements.txt / pyproject.toml |
| Lockfile            | composer.lock | package-lock.json | Gemfile.lock | requirements.lock / poetry.lock |
| Registry            | Packagist   | npmjs.com   | RubyGems    | PyPI        |
| SemVer caret        | `^1.2.3`    | `^1.2.3`    | `~> 1.2`    | `~=1.2.3`   |
| Scripts             | Sim         | Sim         | Não nativo  | Não nativo  |
| Dev vs runtime      | `require-dev` | `devDependencies` | `:development` | extras |

## Links relacionados

- [[Strict-Types]]
- [[Configuracao-PHP-INI]]
- [[../Infra/CI-CD]]
- [[../Infra/Docker-FPM]]
- [[../Infra/PHPStan-Rector]]
- [[../APIs/PSRs]]

---
tags: [php, infra, ci, cd, github-actions]
fase: 7
status: draft
---

# CI / CD com GitHub Actions

## Conceito

**Continuous Integration (CI)** é a prática de mesclar mudanças pequenas com frequência alta numa branch principal, com validação automática a cada merge. **Continuous Delivery (CD)** estende isso até que cada commit que passe nos gates esteja **pronto para deploy** a qualquer momento; **Continuous Deployment** vai um passo além e faz o deploy automático sem intervenção manual. Em PHP, o tooling maduro (Composer, PHPStan, Pest/PHPUnit, Rector, PHP-CS-Fixer) torna esses pipelines acessíveis e baratos.

Um pipeline típico de PHP tem a forma de um **funil de gates**: cada estágio é uma porta que o commit precisa atravessar. Falha em qualquer estágio aborta o resto. A ordem importa por **fail-fast**: estágios baratos e rápidos primeiro (lint, formatação), depois análise estática, depois testes unitários, depois integração, depois build de imagem, depois push para registry, depois deploy. Se o lint falha em 5 segundos, não faz sentido rodar 8 minutos de teste.

O **modelo mental** correto para CI/CD não é "vou rodar testes na nuvem"; é "vou desenhar contratos automatizáveis entre humanos do time". O pipeline codifica regras: "código sem tipos não entra"; "PR sem teste não entra"; "imagem com CVE crítica não entra"; "branch atrasada em mais de N commits do main não entra". Esses contratos eliminam ruído de code review e permitem que humanos olhem para o que importa: design, lógica, trade-offs.

**GitHub Actions** virou o padrão de facto para projetos open-source e médios, mas os mesmos conceitos valem para GitLab CI, CircleCI, Buildkite, Jenkins. O modelo é declarativo via YAML, com **workflows** (arquivos em `.github/workflows/`), **jobs** (executam em runner), **steps** (comandos sequenciais) e **services** (containers auxiliares como MySQL, Redis). Workflows reagem a **events** (push, pull_request, schedule, workflow_dispatch). O runtime é efêmero — cada job começa numa VM limpa.

Estratégias de deploy se dividem em famílias: **rolling** (substitui pods gradualmente — padrão Kubernetes), **blue/green** (duas frotas idênticas, troca o tráfego de uma só vez), **canary** (envia 1%, 5%, 25% do tráfego para a versão nova antes de promover) e **feature flags** (deploy desacoplado de release — código vai pra prod desabilitado, ativado por flag). Cada uma tem trade-offs de complexidade, custo e granularidade de rollback.

## Por que importa

Em CI bem montado, qualquer commit no main produz **artefato pronto para produção em minutos**. Sem CI, "deploy" é manual, longo e medroso — o que reduz frequência, aumenta o tamanho do diff por release e aumenta o risco de cada release. É o ciclo vicioso clássico: medo de deployar → deploys raros → deploys grandes → mais risco → mais medo. CI/CD inverte isso: deploys frequentes → deploys pequenos → risco baixo → confiança alta.

**Cenário 1 — Refatoração ousada**: você quer trocar o ORM ou subir de PHP 8.2 para 8.4. Sem CI, é uma sprint de medo. Com CI rigoroso (PHPStan nível 8 + cobertura > 70% + smoke tests em staging), o feedback é imediato — se quebrou algo, o pipeline avisa em minutos.

**Cenário 2 — Onboarding de júnior**: novo dev abre PR. CI roda lint, análise estática, testes, mostra cobertura. Revisor humano olha só design e clareza, não vírgulas e tipos. Junior aprende com feedback do bot, não com humilhação em review.

**Cenário 3 — Vulnerabilidade publicada**: um CVE crítico cai numa dependência. `composer audit` no CI marca PR vermelho automaticamente; Renovate/Dependabot abre PR de bump; pipeline valida; merge automático se passar. Sem isso, descobre-se a CVE quando o cliente reporta.

**Cenário 4 — Multi-versão**: você mantém biblioteca compatível com PHP 8.2, 8.3 e 8.4. Sem matrix de CI, é loteria. Com matrix, cada PR roda nas três versões em paralelo.

## Exemplo de código PHP

```yaml
# .github/workflows/ci.yml — workflow principal de PR e main
name: CI

on:
  pull_request:
  push:
    branches: [main]

# Cancela runs antigos do mesmo PR ao chegar novo push
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  quality:
    name: Quality (PHP ${{ matrix.php }})
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        php: ['8.3', '8.4']

    services:
      mysql:
        image: mysql:8.4
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: app_test
        ports: ['3306:3306']
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=5s
          --health-timeout=3s
          --health-retries=10
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
        options: --health-cmd="redis-cli ping" --health-interval=5s

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php }}
          extensions: opcache, pdo_mysql, redis, intl, bcmath
          coverage: pcov
          tools: composer:v2

      - name: Get composer cache dir
        id: composer-cache
        run: echo "dir=$(composer config cache-files-dir)" >> "$GITHUB_OUTPUT"

      - name: Cache Composer
        uses: actions/cache@v4
        with:
          path: ${{ steps.composer-cache.outputs.dir }}
          key: ${{ runner.os }}-php${{ matrix.php }}-${{ hashFiles('composer.lock') }}
          restore-keys: ${{ runner.os }}-php${{ matrix.php }}-

      - name: Cache PHPStan result
        uses: actions/cache@v4
        with:
          path: .phpstan-cache
          key: phpstan-${{ matrix.php }}-${{ github.sha }}
          restore-keys: phpstan-${{ matrix.php }}-

      - run: composer install --prefer-dist --no-progress --no-interaction

      - name: Lint (PHP-CS-Fixer)
        run: vendor/bin/php-cs-fixer fix --dry-run --diff --using-cache=no

      - name: Static analysis (PHPStan)
        run: vendor/bin/phpstan analyse --no-progress --error-format=github --memory-limit=512M

      - name: Rector dry-run
        run: vendor/bin/rector process --dry-run

      - name: Composer audit (CVEs)
        run: composer audit --abandoned=report --no-dev

      - name: Run tests
        env:
          DATABASE_URL: mysql://root:root@127.0.0.1:3306/app_test
          REDIS_URL: redis://127.0.0.1:6379
        run: vendor/bin/pest --coverage --min=70 --coverage-clover=coverage.xml

      - name: Upload coverage
        if: matrix.php == '8.4'
        uses: codecov/codecov-action@v4
        with:
          files: coverage.xml
```

```yaml
# .github/workflows/build-and-deploy.yml — release
name: Build and Deploy

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=ref,event=tag
            type=sha,prefix=sha-

      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true
          sbom: true

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - run: ./infra/deploy.sh staging ${{ github.sha }}

  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    # gate manual: required reviewers configurado no environment
    steps:
      - uses: actions/checkout@v4
      - run: ./infra/deploy.sh production ${{ github.sha }}
```

```yaml
# .github/workflows/nightly-security.yml — agendado
name: Nightly Security

on:
  schedule:
    - cron: '0 3 * * *'   # 03:00 UTC todo dia
  workflow_dispatch:

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with: { php-version: '8.4' }
      - run: composer install --no-dev --no-progress
      - run: composer audit --format=json > audit.json
      - name: Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:latest
          severity: CRITICAL,HIGH
          exit-code: 1
```

```yaml
# .github/workflows/phpstan-baseline-guard.yml — bloqueia aumento de erros
name: PHPStan baseline guard

on: pull_request

jobs:
  guard:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: shivammathur/setup-php@v2
        with: { php-version: '8.4' }
      - run: composer install --no-progress
      - name: Count baseline errors
        run: |
          OLD=$(git show origin/main:phpstan-baseline.neon | grep -c '^\s*-\s*message:' || echo 0)
          NEW=$(grep -c '^\s*-\s*message:' phpstan-baseline.neon || echo 0)
          echo "main=$OLD  pr=$NEW"
          if [ "$NEW" -gt "$OLD" ]; then
            echo "::error::Baseline aumentou de $OLD para $NEW. Corrija ao invés de ignorar."
            exit 1
          fi
```

## Estratégias

| Estratégia | Quando aplicar | Custo |
|---|---|---|
| **Cache de composer** | Sempre. Economiza 30s-2min por job | Baixíssimo |
| **Matrix de PHP** | Bibliotecas e apps multi-versão | 2-3x runner-min |
| **Concurrency cancel** | Sempre em PRs. Cancela runs obsoletos | Negativo (economiza) |
| **Service containers** | Quando teste de integração precisa de DB/cache | Médio |
| **Reusable workflows** | Quando há repetição entre repos | Alto setup, baixo manutenção |
| **Self-hosted runners** | Quando volume justifica custo fixo | Alto (manutenção) |
| **Required status checks** | Branch protection do main | Zero |
| **Environments com gate** | Produção | Zero |

**Estratégias de deploy**:

- **Rolling update** (padrão Kubernetes): substitui pods aos poucos. Rollback é deploy reverso. Latência de rollback alta.
- **Blue/green**: duas frotas idênticas, troca DNS/load balancer instantaneamente. Rollback em segundos. Custa 2x infra.
- **Canary**: 5% → 25% → 50% → 100%. Detecta regressão antes de 100%. Requer métricas confiáveis para decisão automatizada.
- **Feature flags**: deploy desacoplado de release. Código vai dark, ligado por flag. Excelente para experiments e migrations gradativas.

## Quando usar / quando não usar

- **CI sempre**: qualquer projeto com mais de uma pessoa contribuindo, ou que vá durar mais de algumas semanas. Custo de configurar < custo do primeiro bug em produção que CI teria pego.
- **CD automático para staging**: em todo merge no main. Permite que QA, Product e demais validem em ambiente real sem ritual.
- **CD automático para prod**: quando o time já tem cultura de feature flags, métricas confiáveis e rollback testado. Comece com gate manual (`environment` com required reviewer) e relaxe conforme confiança.
- **Matrix de PHP**: bibliotecas open-source e aplicações que precisam suportar múltiplas versões. Para SaaS single-tenant em uma versão só, dispensável.
- **Self-hosted runners**: somente quando o custo de minutos GitHub-hosted superar significativamente o custo de manter VMs.
- **Não use** CD automático sem feature flags se a aplicação tem usuários ativos e sem rollback testado.

## Armadilhas comuns

- **CI verde sem testar nada que importa**: cobertura de 5% ainda passa, mas valida só getters. Estabeleça threshold mínimo (`--min` no Pest/PHPUnit) e cobertura por módulo crítico via `pcov` + relatório.
- **`composer install` sem `--no-dev` na imagem final**: vaza PHPUnit, Faker, debugbar para produção. Aumenta superfície de ataque e tamanho. Sempre `--no-dev` no build de imagem.
- **Baseline de PHPStan infinito**: baseline vira tapete debaixo do qual escondem-se erros. Use guard que bloqueia aumento (workflow `phpstan-baseline-guard.yml` acima) e prazo para zerar.
- **CI de 25 minutos**: feedback morto, desenvolvedor faz outra coisa, perde contexto. Paralelize, cacheie, alvo é < 5 minutos para o crítico. Mova testes E2E pesados para noturno.
- **Secrets em logs**: `echo $SECRET` aparece no log do action. Use sempre `::add-mask::` ou configure como `secret` no environment.
- **`GITHUB_TOKEN` com permissões excessivas**: por padrão tem `write` em quase tudo. Use `permissions:` no workflow para mínimo necessário (princípio do menor privilégio).
- **Sem `concurrency cancel-in-progress`**: o time empurra 5 commits seguidos, CI roda 5 vezes em paralelo, fila estoura, billing infla. Sempre configure cancel.
- **Build de imagem sem cache de layers**: cada CI refaz `composer install` do zero, custa 2-3 minutos extras. Use `cache-from: type=gha`.
- **Deploy sem smoke test pós-release**: pipeline marca verde, prod morre porque migration falhou. Sempre rode `curl /health` + alguns endpoints críticos depois do deploy.
- **Sem rollback testado**: na hora do incidente descobre-se que rollback nunca funcionou. Treine rollback no fire drill mensal.

## Links relacionados

- [[Docker-FPM]]
- [[PHPStan-Rector]]
- [[Observabilidade-OWASP]]
- [[Tests/Pest-PHPUnit]]
- [[Fundamentos/Composer]]
- [[Async/RoadRunner-FrankenPHP-Swoole]]

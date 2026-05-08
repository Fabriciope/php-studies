---
tags: [php, infra, ci, cd, github-actions]
fase: 7
status: stub
---

# CI / CD com GitHub Actions

## Conceito

Pipeline mínimo de PHP profissional: **lint** (PHP-CS-Fixer/Pint), **análise estática** (PHPStan/Psalm), **testes** (PHPUnit/Pest), **build** (imagem Docker), **deploy** (registry + rollout).

Cada etapa é uma "gate": falhou → PR não merge, deploy não sai. Sem isso, o repo vira "às vezes funciona".

## Por que importa

Em CI bem montado, qualquer commit no main produz artefato pronto pra produção em minutos. Sem CI, "deploy" é manual, longo e medroso — o que reduz frequência, aumenta tamanho do diff, aumenta risco. Inverte-se o ciclo virtuoso.

## Exemplo de código

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          extensions: opcache, pdo_mysql, redis
          coverage: pcov
          tools: composer:v2
      - name: Cache vendor
        uses: actions/cache@v4
        with:
          path: vendor
          key: ${{ runner.os }}-composer-${{ hashFiles('composer.lock') }}

      - run: composer install --prefer-dist --no-progress
      - run: vendor/bin/phpstan analyse --no-progress --error-format=github
      - run: vendor/bin/php-cs-fixer fix --dry-run --diff
      - run: vendor/bin/phpunit --coverage-text --coverage-clover coverage.xml
      - run: vendor/bin/rector process --dry-run

  build:
    needs: quality
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## Estratégias

- **Cache do composer** entre runs (ganho de minutos).
- **Matrix** para testar PHP 8.3 e 8.4 ao mesmo tempo.
- **Concurrency group** para cancelar runs antigos no mesmo PR (`concurrency: { group: ${{ github.ref }}, cancel-in-progress: true }`).
- **Artifacts** (cobertura, logs) anexados ao run para debugging.
- **Trunk-based** com proteção de branch (status checks obrigatórios + 1 review) > GitFlow para times pequenos.

## Quando usar / quando não usar

- **Sempre** em qualquer projeto que tenha mais de uma pessoa contribuindo.
- **CD automático** para staging em todo merge no main.
- **CD para prod** com **manual gate** (workflow_dispatch ou environment com required reviewer) até o time se sentir confortável.

## Armadilhas comuns

- CI verde **sem testar nada** que importa (cobertura 5% ainda passa). Coloque threshold mínimo (`--min` no phpunit ou via gate).
- Esquecer `composer install --no-dev` na imagem final — leak de dev deps.
- Rodar PHPStan/Psalm em modo "baseline" infinito — baseline vira tapete debaixo do qual escondem-se erros. Reduza baseline com prazo.
- CI rodando 25 minutos — feedback morto. Paralelize, cacheie, reduza para <5 min.

## Links relacionados

- [[Docker-FPM]]
- [[PHPStan-Rector]]
- [[Observabilidade-OWASP]]

---
tags: [php, infra, docker, fpm]
fase: 7
status: stub
---

# Docker multi-stage + FPM tuning

## Conceito

**Multi-stage build**: usar múltiplos `FROM` em um único Dockerfile para separar fase de **build** (composer, dev deps) de fase de **runtime** (só artefato e deps de produção). Imagem final fica enxuta e sem ferramentas que não rodam em prod.

**FPM tuning**: ajustar `pm.*` em `www.conf` para não over/under-provisionar workers — sub-provisionar causa filas, over causa OOM.

## Por que importa

Imagem de 800MB com `composer`, `git`, `xdebug` é convite ao desastre: surface de ataque, deploy lento, custo de storage. Multi-stage corta para <80MB. FPM mal ajustado é razão #1 de "app trava sob carga moderada".

## Exemplo de código

```dockerfile
# === stage 1: deps de produção ===
FROM composer:2 AS vendor
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --no-autoloader --prefer-dist
COPY . .
RUN composer dump-autoload --optimize --classmap-authoritative --no-dev

# === stage 2: runtime ===
FROM php:8.4-fpm-alpine
RUN apk add --no-cache --virtual .deps $PHPIZE_DEPS \
 && docker-php-ext-install opcache pdo_mysql \
 && pecl install redis-6.0.2 \
 && docker-php-ext-enable redis \
 && apk del .deps

COPY infra/php.ini /usr/local/etc/php/conf.d/zz-app.ini
COPY infra/www.conf /usr/local/etc/php-fpm.d/zz-app.conf

WORKDIR /app
COPY --from=vendor --chown=www-data:www-data /app /app

USER www-data
EXPOSE 9000
HEALTHCHECK --interval=10s --timeout=3s \
  CMD php-fpm -t || exit 1
CMD ["php-fpm", "--nodaemonize"]
```

```ini
; www.conf — tuning para 4 vCPU / 4 GB
[www]
listen = 0.0.0.0:9000
pm = dynamic
pm.max_children = 40       ; tetar via stress: rss_max_por_worker (~50MB) * children < RAM
pm.start_servers = 8
pm.min_spare_servers = 4
pm.max_spare_servers = 12
pm.max_requests = 500       ; recicla worker — evita leak
request_terminate_timeout = 30s
catch_workers_output = yes
decorate_workers_output = no

clear_env = no
; healthcheck
pm.status_path = /status
ping.path = /ping
```

## Cálculo de `pm.max_children`

```
max_children = (RAM_total - RAM_so - RAM_outros_serviços) / RSS_médio_por_worker
```

Medindo: rode app sob carga, veja `ps aux | grep php-fpm` para RSS médio. Se cada worker usa 80MB e a VM tem 3GB livres → 3000/80 ≈ 37 workers seguros.

## Quando usar / quando não usar

- **Multi-stage** sempre. Não há motivo para imagem monolítica em 2026.
- **Alpine** geralmente OK; mas há corner cases (musl vs glibc) — em domínio com lib C exótica, prefira `bookworm-slim`.
- **`pm = static`** quando a carga é constante e previsível (background workers). `dynamic` para web variável.

## Armadilhas comuns

- `composer install` no estágio final — vendor leva 200MB, devs deps incluídas.
- Rodar como `root` — vulnerabilidade. Use `USER www-data`.
- Esquecer `clear_env = no` quando seu app lê env vars — vão estar vazias.
- `pm.max_children` muito alto: OOM-killer mata workers, requests morrem com 502. Calcule.
- `pm.max_requests = 0` (default) — sem reciclo, leak vence. Use 500–2000.

## Links relacionados

- [[Fundamentos/Configuracao-PHP-INI]]
- [[Performance/OPcache-JIT-Preloading]]
- [[CI-CD]]

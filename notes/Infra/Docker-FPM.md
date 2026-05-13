---
tags: [php, infra, docker, fpm]
fase: 7
status: draft
---

# Docker multi-stage + FPM tuning

## Conceito

Containerizar PHP em produção significa orquestrar três decisões interligadas: **qual imagem base** (cli, fpm, alpine, debian-slim, frankenphp), **como o build é fatiado** (multi-stage para minimizar superfície e tamanho final) e **como o php-fpm é parametrizado** (workers, recycling, timeouts). Errar qualquer uma das três se manifesta de forma diferente: imagem inchada gera deploys lentos e CVEs latentes; build monolítico vaza ferramentas como `composer`, `git` e `xdebug` para produção; FPM mal ajustado derruba a API sob carga moderada com filas, 502 e OOM-killer.

O **modelo mental** mais útil é pensar em duas máquinas distintas dentro do mesmo Dockerfile: uma "máquina de build" rica em ferramentas (composer, extensões de compilação, código-fonte completo) e uma "máquina de runtime" minimalista que recebe somente o artefato pronto (vendor otimizado, configurações, binários PHP). O `COPY --from=<stage>` é a ponte entre elas. Em prod, a máquina de runtime nem deveria ter `composer` instalado — se precisar dele em runtime, o desenho está errado.

As **imagens oficiais** mais usadas são `php:8.4-fpm`, `php:8.4-fpm-alpine`, `php:8.4-cli` e `php:8.4-cli-alpine`. A variante `fpm` expõe um daemon FastCGI que aguarda requisições via socket TCP (9000) ou Unix; precisa de um reverse proxy na frente (NGINX, Caddy, Traefik) que fala FastCGI upstream. A variante `cli` é o que você usa para workers, cron, comandos artisan/console e testes. **Não existe FROM scratch para PHP**: o binário precisa de libc, libxml, libssl, zlib e dezenas de outras libs C dinâmicas — daí o uso de bases minimalistas como Alpine (musl libc, ~6MB) ou Debian slim (~25MB).

O **cache de layers** do Docker é o que torna o build incremental: cada instrução cria uma layer e a próxima só é refeita se algum input mudou. A ordem das instruções importa: copie `composer.json` e `composer.lock` antes do código da aplicação, rode `composer install`, e só depois `COPY . .`. Assim, mudar uma classe não invalida a layer de vendor — economiza minutos em CI. Essa técnica é conhecida como **dependency caching** e é universal (npm, pip, cargo seguem o mesmo padrão).

Uma alternativa recente e relevante é **FrankenPHP**: um único binário que embute PHP + servidor HTTP (Caddy) + worker mode, eliminando a separação NGINX/FPM. Para apps pequenos a médios, simplifica drasticamente a topologia; mas o ecossistema ainda é menos maduro que o par NGINX+FPM e algumas extensões podem ter atrito.

## Por que importa

Imagem de 800MB carregando `composer`, `git`, `xdebug`, headers de desenvolvimento e ferramentas de compilação é convite ao desastre: aumenta a **superfície de ataque** (cada binário extra é potencial CVE), encarece o registry, faz cada deploy demorar mais para pull, lota o disco dos nós do cluster e atrasa autoscaling. Uma imagem multi-stage bem feita fica entre 60MB e 120MB e sobe em segundos.

**FPM mal ajustado** é a causa número um de "minha aplicação trava quando chegam 50 reqs/s". Os sintomas típicos: latência p99 explode, requests ficam pendurados na fila do FPM, NGINX começa a retornar 502 Bad Gateway, e o OOM-killer começa a abater workers aleatoriamente. Quase sempre a raiz é `pm.max_children` definido por intuição ("um número redondo, tipo 100") sem cálculo de RSS por worker.

Outro cenário recorrente: **memory leak silencioso** em código de aplicação ou em extensão de terceiros. Sem `pm.max_requests` configurado, o worker nunca recicla e cresce até o limite de memória do container. Reiniciar a cada N requisições é a forma barata de conviver com leaks até descobrir a raiz.

Há também o problema de **observabilidade**: um container sem `pm.status_path` ativado é cego — você não consegue exportar métricas para Prometheus, não sabe quantos workers estão ocupados, quantos idle, qual o tamanho da fila. Em incidente, isso significa adivinhar.

## Exemplo de código PHP

```dockerfile
# === stage 1: vendor de produção ===
FROM composer:2 AS vendor
WORKDIR /app
# Copia só os manifestos primeiro para maximizar cache
COPY composer.json composer.lock ./
RUN composer install \
    --no-dev \
    --no-scripts \
    --no-autoloader \
    --prefer-dist \
    --no-interaction
# Agora o código entra; dump-autoload otimizado vira a última etapa
COPY . .
RUN composer dump-autoload --optimize --classmap-authoritative --no-dev

# === stage 2: runtime enxuto ===
FROM php:8.4-fpm-alpine AS runtime

# Extensões necessárias. Instaladas como pacote temporário e removidas no fim
RUN apk add --no-cache --virtual .build-deps $PHPIZE_DEPS linux-headers \
 && apk add --no-cache icu-libs libpq libzip oniguruma \
 && docker-php-ext-install -j$(nproc) opcache pdo_mysql pdo_pgsql intl bcmath \
 && pecl install redis-6.0.2 apcu-5.1.23 \
 && docker-php-ext-enable redis apcu \
 && apk del .build-deps \
 && rm -rf /tmp/* /var/cache/apk/*

# Configs vão para conf.d (prefixo zz- garante ordem após defaults)
COPY infra/php.ini       /usr/local/etc/php/conf.d/zz-app.ini
COPY infra/opcache.ini   /usr/local/etc/php/conf.d/zz-opcache.ini
COPY infra/www.conf      /usr/local/etc/php-fpm.d/zz-app.conf

WORKDIR /app
# Copia artefato pronto do stage de build, já com ownership correto
COPY --from=vendor --chown=www-data:www-data /app /app

# Não rode como root
USER www-data
EXPOSE 9000

# Healthcheck: testa o parser de config + ping endpoint
HEALTHCHECK --interval=10s --timeout=3s --start-period=15s --retries=3 \
  CMD php-fpm -t && SCRIPT_NAME=/ping SCRIPT_FILENAME=/ping \
      REQUEST_METHOD=GET cgi-fcgi -bind -connect 127.0.0.1:9000 || exit 1

CMD ["php-fpm", "--nodaemonize"]
```

```ini
; infra/www.conf — tuning para nó de 4 vCPU / 4 GB RAM
[www]
user  = www-data
group = www-data

listen = 0.0.0.0:9000
listen.backlog = 511

; ----- estratégia de pool -----
; dynamic   → workers entre min/max, escalam conforme demanda. Default web.
; static    → número fixo de workers. Use quando carga é constante.
; ondemand  → cria worker só quando chega request. Use para serviços raros.
pm = dynamic
pm.max_children      = 40    ; teto absoluto: ver cálculo abaixo
pm.start_servers     = 8     ; spawnados no boot
pm.min_spare_servers = 4     ; mínimo de idle prontos
pm.max_spare_servers = 12    ; idle acima disso são terminados
pm.process_idle_timeout = 30s
pm.max_requests = 500        ; recicla worker após N reqs (antídoto a leak)

; ----- timeouts -----
request_terminate_timeout = 30s    ; mata request que excede (PHP > NGINX)
request_slowlog_timeout   = 5s
slowlog = /proc/self/fd/2

; ----- saída e logs -----
catch_workers_output    = yes
decorate_workers_output = no
access.log = /proc/self/fd/2
access.format = "%R - %u %t \"%m %r\" %s %f %{mili}d %{kilo}M %C%%"

; ----- ambiente -----
clear_env = no               ; preserva env do container (necessário para 12-factor)

; ----- observabilidade -----
pm.status_path = /status     ; expõe métricas para Prometheus exporter
ping.path      = /ping
ping.response  = pong
```

```nginx
# infra/nginx.conf — fragmento do server bloco
server {
    listen 80;
    root  /app/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass   php-fpm:9000;     # nome do service no compose
        fastcgi_index  index.php;
        include        fastcgi_params;
        fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;

        fastcgi_read_timeout 30s;
        fastcgi_buffers 16 16k;
        fastcgi_buffer_size 32k;
    }

    # Expor status do FPM apenas em rede interna
    location ~ ^/(status|ping)$ {
        allow 10.0.0.0/8;
        deny  all;
        fastcgi_pass php-fpm:9000;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $fastcgi_script_name;
    }
}
```

```yaml
# docker-compose.yml — ambiente de desenvolvimento
services:
  php-fpm:
    build:
      context: .
      target: runtime
    volumes:
      - ./:/app:cached
      - ./infra/php.dev.ini:/usr/local/etc/php/conf.d/zz-dev.ini:ro
    environment:
      APP_ENV: local
      DATABASE_URL: mysql://app:app@mysql:3306/app
    depends_on: [mysql, redis]

  nginx:
    image: nginx:1.27-alpine
    ports: ["8080:80"]
    volumes:
      - ./:/app:cached
      - ./infra/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on: [php-fpm]

  mysql:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: app
      MYSQL_USER: app
      MYSQL_PASSWORD: app
    volumes: [mysql-data:/var/lib/mysql]

  redis:
    image: redis:7-alpine

volumes:
  mysql-data:
```

## Cálculo de `pm.max_children`

A fórmula prática:

```
pm.max_children = (RAM_disponível_container - RAM_reservada_so_e_extras) / RSS_médio_por_worker
```

**Como medir o RSS por worker em produção**:

```bash
ps -ylC php-fpm --sort:rss | awk '{ s += $8 } END { print s/NR/1024 " MB médio por worker" }'
```

Exemplo: container com limit de 3 GB, sistema reserva ~300 MB para SO/buffers, RSS médio observado é 80 MB por worker:

```
(3000 - 300) / 80 = 33.75  →  pm.max_children = 32
```

Sempre arredonde para baixo e deixe folga. Validar com `wrk`, `k6` ou `ab` antes de subir para produção.

## Quando usar / quando não usar

- **Multi-stage build**: sempre. Não há cenário moderno onde imagem monolítica seja preferível.
- **Alpine (musl libc)**: padrão para 80% dos casos. Evite quando depende de libs C exóticas (oracle instantclient, ML libs) — prefira `bookworm-slim`.
- **`pm = dynamic`**: padrão para web com tráfego variável.
- **`pm = static`**: workers de filas, jobs de longa duração, carga previsível.
- **`pm = ondemand`**: serviços raramente acessados (admin interno, cron HTTP-trigger).
- **FrankenPHP**: ótimo para apps pequenos e médios, worker mode (PHP residente em memória) é game changer para latência. Avalie maturidade das extensões antes.
- **Não use** `php:8.4-cli` para servir web: ele não tem FPM e o servidor builtin (`php -S`) não é para produção.

## Armadilhas comuns

- **`composer install` rodando no stage final**: vendor inclui dev deps (PHPUnit, Faker, Mockery), imagem incha 200MB+ e ferramentas de teste viram superfície de ataque. Faça o install no stage de build e copie só o resultado.
- **Rodar como `root`**: vulnerabilidade séria — uma RCE via upload mal validado vira escalada de privilégio no container. Sempre `USER www-data` (ou criado explicitamente).
- **Esquecer `clear_env = no`**: por padrão FPM limpa o ambiente antes de spawnar workers, então `getenv('DATABASE_URL')` retorna `false` mesmo com a env definida no compose/k8s. Sintoma clássico: "funciona local, não funciona no container".
- **`pm.max_children` no chute**: define-se 200, container tem 1GB, cada worker pesa 100MB → OOM-killer abate workers, requests morrem com 502. Calcule sempre.
- **`pm.max_requests = 0`** (default): worker nunca recicla; qualquer leak (mesmo de extensão de terceiros) cresce até travar. Use entre 500 e 2000.
- **Misturar `COPY . .` antes do `composer install`**: invalida o cache do vendor a cada mudança de código, build vai de 30s para 5 minutos. Sempre copie manifestos primeiro.
- **Sem `request_terminate_timeout`**: query lenta ou loop infinito segura o worker indefinidamente; FPM acaba com todos workers ocupados e a aplicação para. Configure agressivo (15-30s para web).
- **Healthcheck que só testa porta TCP**: a porta pode estar aberta com FPM em deadlock. Teste `/ping` real ou `php-fpm -t`.
- **Logs em arquivo dentro do container**: viola 12-factor, perde logs em restart, lota disco. Direcione para `/proc/self/fd/2` (stderr) e deixe o orquestrador coletar.

## Links relacionados

- [[Fundamentos/Configuracao-PHP-INI]]
- [[Performance/OPcache-JIT-Preloading]]
- [[CI-CD]]
- [[Observabilidade-OWASP]]
- [[Performance/Workers-Filas-Async]]
- [[Async/RoadRunner-FrankenPHP-Swoole]]

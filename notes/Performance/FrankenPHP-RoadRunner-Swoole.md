---
tags: [php, performance, runtime, frankenphp, roadrunner, swoole]
fase: 6
status: draft
---

# FrankenPHP / RoadRunner / Swoole

## Conceito

O modelo histórico de execução do PHP em produção é o **shared-nothing** via PHP-FPM (FastCGI Process Manager): para cada request HTTP, o webserver (Nginx/Apache) repassa via socket FastCGI a um worker PHP-FPM, que executa o script do zero — incluindo bootstrap do framework, montagem de container DI, abertura de conexões — responde e descarta tudo. A memória do processo é "esquecida" entre requests (do ponto de vista lógico; o processo continua vivo mas o estado é resetado pelo término do script). Isso é simples, robusto, **à prova de leak**: mesmo que código vaze memória ou estado, o "fim de request" limpa. É o que tornou PHP famoso por estabilidade desde os anos 2000.

O preço é o **custo de bootstrap repetido**. Em uma aplicação Symfony média, o bootstrap consome 30-80ms por request — autoload, instanciação de container, leitura de config, conexão DB. OPcache+Preloading amenizam, mas não eliminam (objetos ainda são reconstruídos a cada request). Em apps "leves" (microservices CRUD), isso significa que 60-80% do tempo de cada request é overhead repetitivo.

Runtimes alternativos invertem o modelo: **long-running workers** que mantêm o processo PHP **vivo entre requests**. Bootstrap acontece uma vez, no boot do worker; cada request reusa container DI, conexões DB, JIT aquecido, objetos imutáveis em memória. Esse padrão é trivial em Node.js, Go, Python (Gunicorn workers), Java — mas era exótico em PHP até recentemente.

**FrankenPHP** (Kévin Dunglas, 2022+) é o mais novo. Servidor web Caddy escrito em Go com PHP embutido como módulo (não há processo PHP separado — Caddy é o webserver E o runtime). Single-binary, suporte nativo a HTTP/2, HTTP/3, WebSockets, Early Hints (103), TLS automático via Let's Encrypt. Tem dois modos: **classic** (FPM-like, isolated requests) e **worker mode** (long-running, ganhos máximos). Adoption curve mais suave que RoadRunner.

**RoadRunner** (Spiral Scout, 2018+) é um servidor de aplicação em Go que orquestra um pool de workers PHP comunicando via PSR-7 sobre goridge (RPC binário). Maduro, ecossistema rico: plugins para **jobs** (filas embutidas — Redis, AMQP, SQS), **KV** (cache), **gRPC**, **temporal.io workflows**, **websockets**. A combinação RoadRunner + Spiral Framework forma um stack PHP "Node-like" completo.

**Swoole / OpenSwoole** (2012+) é radicalmente diferente: uma **extensão C** que adiciona ao PHP um event loop (similar a libuv do Node.js), **corrotinas** (não as Fibers do PHP 8.1 — Swoole tem coroutines próprias desde 2018), e clientes async para MySQL, Redis, HTTP, gRPC. Permite escrever PHP com sintaxe blocante que internamente é cooperativamente concorrente — um worker pode processar 10000 requests "simultâneas" em uma única thread. Poderoso, mas exige código consciente: variáveis globais, conexões compartilhadas, estado mutável são minas terrestres.

**Distinção Fibers vs Swoole Coroutines**: Fibers (PHP 8.1+, oficial) são primitivos de baixo nível — você precisa de scheduler/event loop externo (Amphp/Revolt) para usá-los praticamente. Swoole Coroutines vêm com scheduler integrado e clientes async prontos — mais "pilhas incluídas", mas só funciona dentro do runtime Swoole.

| Critério                | FPM           | FrankenPHP        | RoadRunner       | Swoole               |
| ----------------------- | ------------- | ----------------- | ---------------- | -------------------- |
| Modelo                  | shared-nothing | worker pool      | worker pool      | event loop + corrotinas |
| Setup                   | trivial       | binário único     | binário Go + cfg | extensão C compilada |
| Distribuição            | apt/yum       | docker/binary     | docker/binary    | pecl                 |
| RPS típico (Hello)      | ~3k           | ~15k              | ~14k             | ~30k                 |
| RPS típico (Symfony)    | ~800          | ~3500             | ~3200            | ~5000                |
| Concorrência por worker | 1 (request)   | 1 (request)       | 1 (request)      | N (coroutines)       |
| Async I/O nativo        | não           | não               | não              | sim                  |
| HTTP/2 e HTTP/3         | via reverse proxy | nativo        | depende          | sim                  |
| Hot reload              | trivial       | via worker recycle | via worker recycle | via worker recycle |
| Maturidade ecossistema  | máxima        | jovem (~2023)     | madura           | madura               |
| Risco de leak           | baixo         | alto              | alto             | altíssimo            |
| Frameworks oficialmente suportados | todos | Symfony, Laravel | Spiral, Symfony, Laravel | Hyperf, Swoft |

## Por que importa

Benchmarks reais (Symfony Demo, mesma máquina, 100 conexões concorrentes):

| Setup                                      | RPS    | p50   | p95   | Memória/worker |
| ------------------------------------------ | ------ | ----- | ----- | -------------- |
| FPM + OPcache                              | 850    | 110ms | 180ms | 40MB           |
| FPM + OPcache + Preload + JIT              | 1100   | 88ms  | 150ms | 55MB           |
| FrankenPHP worker mode                     | 3500   | 27ms  | 45ms  | 90MB           |
| RoadRunner                                 | 3200   | 30ms  | 48ms  | 85MB           |
| Swoole (coroutines + async DB)             | 6800   | 14ms  | 28ms  | 120MB          |

Ganho de 3-4x em apps complexas, 6-8x em apps simples. **Mas o ganho real depende do peso do bootstrap**: app com bootstrap leve (10ms) tem ganho menor; app com bootstrap pesado (Symfony + Doctrine + 30 bundles, ~80ms) tem ganho maior.

Para Swoole, o ganho extra vem do async I/O: durante I/O, um worker continua processando outras requests. Em app que faz 5 chamadas HTTP externas em sequência (200ms cada = 1s), Swoole pode rodar paralelo (200ms total). FPM e RoadRunner não — cada worker bloqueia.

Contraponto importante: **complexidade operacional**. FPM é "instale e esqueça". Worker runtimes exigem: monitoração de memória (workers reciclam), cuidado com estado (singletons, conexões), gestão de signals (graceful shutdown), debug mais complexo (stack trace pula entre requests). Time pequeno: prefira FPM até medir necessidade real.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// === FrankenPHP worker mode — entry point ===
// public/index.php

require __DIR__ . '/../vendor/autoload.php';

$kernel = new App\Kernel($_SERVER['APP_ENV'] ?? 'prod', (bool) ($_SERVER['APP_DEBUG'] ?? false));
$kernel->boot();

$handler = static function () use ($kernel): void {
    $request  = Symfony\Component\HttpFoundation\Request::createFromGlobals();
    $response = $kernel->handle($request);
    $response->send();
    $kernel->terminate($request, $response);
};

// Loop: cada iteração é uma request HTTP
$maxRequests = (int) ($_SERVER['MAX_REQUESTS'] ?? 1000);
for ($n = 0; $n < $maxRequests; ++$n) {
    $keepRunning = \frankenphp_handle_request($handler);

    // Hygiene entre requests
    gc_collect_cycles();

    if (!$keepRunning) {
        break;
    }
}
```

```php
<?php
declare(strict_types=1);

// === RoadRunner worker — entry point ===
// worker.php

require __DIR__ . '/vendor/autoload.php';

use Spiral\RoadRunner\Worker;
use Spiral\RoadRunner\Http\PSR7Worker;
use Nyholm\Psr7\Factory\Psr17Factory;

$factory = new Psr17Factory();
$worker  = Worker::create();
$psr7    = new PSR7Worker($worker, $factory, $factory, $factory);

$kernel = require __DIR__ . '/bootstrap.php'; // monta container UMA vez

while (true) {
    try {
        $request = $psr7->waitRequest();
        if ($request === null) {
            break; // shutdown gracefully
        }
    } catch (\Throwable $e) {
        $psr7->respond(new Nyholm\Psr7\Response(400));
        continue;
    }

    try {
        $response = $kernel->handle($request);
        $psr7->respond($response);
    } catch (\Throwable $e) {
        $psr7->respond(new Nyholm\Psr7\Response(500, [], 'Internal Server Error'));
        $worker->error((string) $e);
    } finally {
        // CRÍTICO: resetar estado por request em runtime persistente
        $kernel->reset();
    }
}
```

```yaml
# RoadRunner .rr.yaml
version: "3"

server:
    command: "php worker.php"
    relay: pipes

http:
    address: 0.0.0.0:8080
    middleware: ["headers", "static", "gzip"]
    pool:
        num_workers: 8
        max_jobs: 1000           # recicla worker depois de N requests
        supervisor:
            ttl: 0
            idle_ttl: 0
            max_worker_memory: 256   # MB; mata worker se exceder
            exec_ttl: 60s

jobs:
    consume: ["emails", "reports"]
    pipelines:
        emails:
            driver: redis
            config:
                priority: 10
        reports:
            driver: amqp
            config:
                routing_key: "reports"
```

```php
<?php
declare(strict_types=1);

// === Swoole — HTTP server com coroutines + async ===
use Swoole\Http\Server;
use Swoole\Http\Request;
use Swoole\Http\Response;
use Swoole\Coroutine\MySQL;

$http = new Server('0.0.0.0', 9501);
$http->set([
    'worker_num'        => 4,
    'task_worker_num'   => 2,
    'max_request'       => 1000,
    'enable_coroutine'  => true,
]);

// Container/dependências montadas no onStart, compartilhadas por workers
$http->on('workerStart', function (Server $server, int $workerId) use (&$container) {
    $container = require __DIR__ . '/bootstrap.php';
    // Conexões abertas aqui são per-worker; cuidado com pool
});

$http->on('request', function (Request $req, Response $res) use (&$container) {
    // Cada request roda em coroutine; durante I/O outras requests progridem
    go(function () use ($req, $res, $container) {
        $db = new MySQL();
        $db->connect(['host' => 'mysql', 'user' => 'app', 'password' => 'x', 'database' => 'app']);

        // I/O async: durante essa query, worker atende outras requests
        $result = $db->query("SELECT * FROM users WHERE id = " . (int) $req->get['id']);

        $res->header('Content-Type', 'application/json');
        $res->end(json_encode($result[0] ?? null));
    });
});

$http->start();
```

```dockerfile
# === FrankenPHP — Dockerfile típico ===
FROM dunglas/frankenphp:1-php8.3-alpine

ENV SERVER_NAME="https://app.example.com"
ENV FRANKENPHP_CONFIG="worker ./public/index.php"

COPY --chown=www-data . /app
WORKDIR /app

RUN install-php-extensions \
        opcache pdo_mysql redis intl bcmath

COPY php.ini /usr/local/etc/php/conf.d/app.ini

# Caddy config: aproveita HTTP/3, TLS auto
COPY Caddyfile /etc/caddy/Caddyfile

EXPOSE 80 443 443/udp
```

## Quando usar / quando não usar

- **FPM** — default seguro. Time sem experiência operacional em workers; app de baixa/média carga; equilíbrio entre simplicidade e performance.
- **FPM + OPcache + Preload + JIT** — primeiro alvo de tuning antes de mudar runtime. Ganho 30-50% sem trocar arquitetura.
- **FrankenPHP** — projetos novos, prioridade em simplicidade operacional (single binary), uso de HTTP/3, suporte oficial a Symfony/Laravel, time pequeno. Comunidade crescendo rápido.
- **RoadRunner** — quando precisa de jobs nativos integrados, gRPC, workflows (Temporal), múltiplos tipos de worker (HTTP + jobs + cron) em um único orquestrador, ecossistema Spiral.
- **Swoole** — gateways API, proxies, sistemas com alta concorrência I/O (chat, websocket, agregadores que fazem dezenas de chamadas paralelas). Exige time experiente; código precisa ser escrito conscientemente.
- **Não migre** para worker runtime apenas por "performance" se profile não mostra bootstrap como hot — pode não dar ROI.
- **Não use** Swoole sem revisão completa do código se app tinha estado global, ORM com identity map, singletons mutáveis. Migração pode levar semanas.
- **Não use** FrankenPHP em ambiente onde TLS automático interfere com infraestrutura existente (load balancers que terminam TLS).

## Armadilhas comuns

1. **Estado global / singletons mutáveis.** `User::$current` setado no início da request continua setado na próxima — usuário A vê dados do usuário B. *Por quê*: em FPM o processo morre; em worker persistente, variáveis estáticas e singletons perduram. Audite todo `static $x` e singleton container; resete entre requests.

2. **Conexões PDO em estado errado entre requests.** Transação foi aberta na request anterior e não fechada (exception não tratada) — próxima request herda conexão "suja", queries falham com "There is already an active transaction". *Por quê*: pool de conexões persistente sem reset. Sempre `try/finally` com `rollBack()` em catch, e idealmente devolver conexão "limpa" ao pool.

3. **Memória crescendo sem limite.** Worker processa 10k requests, RSS sobe de 80MB para 2GB, OOM. *Por quê*: ORM com identity map global, listeners acumulando, caches em memória sem TTL. Mitigações: `max_jobs`/`max_requests` para reciclar workers periodicamente, `max_worker_memory` para matar se exceder limite, `gc_collect_cycles()` periódico, `$em->clear()` entre requests.

4. **Doctrine EntityManager fechado.** Uma exception fecha o EM; próxima request com mesmo worker recebe `EntityManager is closed`. *Por quê*: EM tem ciclo de vida amarrado a request, mas worker persistente o reusa. Resetar via `$registry->resetManager()` ou recriar EM no início de cada request.

5. **Logger acumulando handlers a cada request.** Em FPM, `Monolog::pushHandler` é chamado uma vez por request e morre; em worker, acumula a cada request → memória cresce, logs duplicam. *Por quê*: bootstrap não foi separado de "per-request". Mova setup de logger para o boot único.

6. **`$_SESSION` e cookies com Swoole.** Funções nativas de sessão do PHP (`session_start`) não funcionam em Swoole — não há "request global". Precisa de middleware de sessão. *Por quê*: Swoole abstrai HTTP, mas APIs globais do PHP esperam SAPI tradicional.

7. **Esquecer `gc_collect_cycles()`.** PHP coleta automaticamente referências cíclicas só quando buffer enche; em worker long-running, podem demorar muito. *Por quê*: ciclos referenciais (A → B → A) não são coletados por refcount sozinho. Force coleta a cada N requests.

8. **Debug confuso por re-execução.** Erro misterioso "variável null" só após 500 requests — primeira request roda OK, alguma request entre rasgou estado. *Por quê*: bugs por estado compartilhado são intermitentes e dependem da ordem de requests. Use logs por request-id, ferramentas de debug remoto (Xdebug com cuidado), reproduza com testes de carga concorrente.

## Links relacionados

- [[OPcache-JIT-Preloading]]
- [[Fibers-N1]]
- [[Infra/Docker-FPM]]
- [[Filas]]
- [[Profiling]]
- [[Cache-PSR-6-16]]

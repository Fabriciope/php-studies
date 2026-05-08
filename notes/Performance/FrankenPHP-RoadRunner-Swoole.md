---
tags: [php, performance, runtime, frankenphp, roadrunner, swoole]
fase: 6
status: stub
---

# FrankenPHP / RoadRunner / Swoole

## Conceito

Runtimes alternativos ao FPM clássico. Em FPM, cada request faz **bootstrap** (autoload, container, conexão DB) — mesmo com OPcache, há overhead a cada chamada. Estes runtimes mantêm o processo PHP **vivo entre requests**:

- **FrankenPHP** — servidor Caddy + PHP embutido. Single-binary, suporte a HTTP/3, modo "worker" semelhante ao RoadRunner. O mais novo e simples de adotar.
- **RoadRunner** — servidor em Go, workers em PHP via PSR-7. Maduro, plugins (jobs, KV, gRPC). Excelente.
- **Swoole / OpenSwoole** — extensão C com event loop, corrotinas, async I/O. Mais poderoso, mais complexo, exige código consciente de estado compartilhado.

## Por que importa

A maior parte do tempo de uma request "leve" em FPM é bootstrap. Em runtime persistente, ganho típico é **2–5x RPS** sem mudar uma linha de domínio. Trade-off: agora **vazamento de memória/estado é problema** (em FPM o processo morre a cada request e limpa).

## Exemplo de código PHP

```php
<?php
// FrankenPHP em modo worker — index.php é o ponto de entrada
declare(strict_types=1);
require __DIR__ . '/vendor/autoload.php';

$app = require __DIR__ . '/bootstrap.php'; // monta container UMA vez

$handler = static function () use ($app): void {
    $req = ServerRequestFactory::fromGlobals();
    $res = $app->handle($req);
    Emitter::emit($res);
};

// loop: cada iteração é uma request
for ($n = 0; $n < 1000; $n++) {
    $keepRunning = \frankenphp_handle_request($handler);
    gc_collect_cycles();             // boa prática
    if (!$keepRunning) break;
}
```

```php
<?php
// RoadRunner — mesma ideia, API ligeiramente diferente
$worker = Spiral\RoadRunner\Worker::create();
$psr7   = new Spiral\RoadRunner\Http\PSR7Worker(
    $worker, new ServerRequestFactory(), new StreamFactory(), new UploadedFileFactory()
);

while ($req = $psr7->waitRequest()) {
    try {
        $res = $app->handle($req);
        $psr7->respond($res);
    } catch (\Throwable $e) {
        $psr7->getWorker()->error((string) $e);
    }
}
```

## Comparação

| Critério           | FPM           | FrankenPHP        | RoadRunner       | Swoole              |
| ------------------ | ------------- | ----------------- | ---------------- | ------------------- |
| Setup              | trivial       | binário único     | binário Go + cfg | extensão C          |
| RPS típico (hello) | baixo         | alto              | alto             | altíssimo           |
| Concorrência       | processo/req  | worker pool       | worker pool      | corrotinas async    |
| Async I/O          | não           | não (sync no PHP) | não              | sim (event loop)    |
| Maturidade         | máxima        | jovem             | madura           | madura              |
| Risk de leak       | baixo         | alto              | alto             | alto                |

## Quando usar / quando não usar

- **FPM** — default seguro. App sem requisito de performance extrema.
- **FrankenPHP** — projeto novo onde simplicidade importa. Single binary.
- **RoadRunner** — quando precisa de gRPC, jobs nativos, workers heterogêneos.
- **Swoole** — só com time experiente, especialmente para I/O intensivo (proxies, gateways).

## Armadilhas comuns

- **Estado global** ou singletons que mantêm dados de request anterior. Em FPM "funciona"; em worker persistente, vaza dados entre usuários.
- Conexões PDO em estado errado (transação aberta) entre requests — corrupção silenciosa.
- Memória cresce sem fim — adicione `max_requests` por worker (reciclar a cada N requests).
- ORM com identity map global — explode em horas. Resete a cada request.

## Links relacionados

- [[OPcache-JIT-Preloading]]
- [[Fibers-N1]]
- [[Infra/Docker-FPM]]

---
tags: [php, api, psr, http]
fase: 5
status: stub
---

# PSRs essenciais (7/15/17/3/11/18)

## Conceito

PSRs (PHP Standards Recommendations) são contratos publicados pelo PHP-FIG. Em APIs maduras, seis importam:

- **PSR-7** — `RequestInterface`/`ResponseInterface` (HTTP messages)
- **PSR-17** — `RequestFactoryInterface`/`StreamFactoryInterface` (criar mensagens)
- **PSR-15** — `MiddlewareInterface`/`RequestHandlerInterface` (servidor HTTP)
- **PSR-3** — `LoggerInterface`
- **PSR-11** — `ContainerInterface`
- **PSR-18** — `ClientInterface` (cliente HTTP)

## Por que importa

Adotar PSRs significa que sua app pode usar middleware da Symfony, logger da Monolog, container do PHP-DI e cliente Guzzle — todos plugáveis. É a base do PHP "moderno e portátil".

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

use Psr\Http\Message\{ServerRequestInterface, ResponseInterface};
use Psr\Http\Server\{MiddlewareInterface, RequestHandlerInterface};
use Psr\Log\LoggerInterface;

// PSR-15 middleware (logging estruturado)
final readonly class RequestLogger implements MiddlewareInterface {
    public function __construct(private LoggerInterface $logger) {}

    public function process(
        ServerRequestInterface $req,
        RequestHandlerInterface $next,
    ): ResponseInterface {
        $start = hrtime(true);
        $traceId = $req->getHeaderLine('X-Trace-Id') ?: bin2hex(random_bytes(8));

        $this->logger->info('http.request', [
            'method' => $req->getMethod(),
            'path' => $req->getUri()->getPath(),
            'trace_id' => $traceId,
        ]);

        $res = $next->handle($req->withAttribute('trace_id', $traceId));
        $ms  = (hrtime(true) - $start) / 1e6;

        $this->logger->info('http.response', [
            'status' => $res->getStatusCode(),
            'duration_ms' => $ms,
            'trace_id' => $traceId,
        ]);
        return $res->withHeader('X-Trace-Id', $traceId);
    }
}
```

## Quando usar / quando não usar

- **PSR-7/15** — sempre em APIs novas. Compatibilidade com qualquer ecossistema.
- **PSR-3** — sempre. Monolog é o de fato; código depende da interface.
- **PSR-11** — sempre. Container concreto pode trocar; código depende só da interface.
- **PSR-18** — quando consumir HTTP externo. Cliente concreto (Guzzle, Symfony HttpClient, cURL) vai atrás.
- **Não exija** PSRs em código interno simples (workers de fila pequenos, scripts) — adicione quando o ecossistema do middleware compensa.

## Armadilhas comuns

- PSR-7 messages são **imutáveis**. `$res->withStatus(404)` retorna **nova** Response — esquecer isso é o bug clássico.
- Misturar PSR-7 com modelo do framework (Symfony Request) — escolha um boundary claro.
- Logger PSR-3 com placeholders interpolados manualmente (`"user $id"`) em vez de `$logger->info('user {id}', ['id' => $id])`. O segundo permite log estruturado.

## Links relacionados

- [[REST-Maduro]]
- [[Versionamento-Idempotencia]]
- [[Infra/Observabilidade-OWASP]]

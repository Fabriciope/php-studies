---
tags: [php, api, psr, http]
fase: 5
status: draft
---

# PSRs essenciais (7/15/17/3/11/18)

## Conceito

PSRs (PHP Standards Recommendations) são contratos publicados pelo PHP-FIG (PHP Framework Interoperability Group), um consórcio informal nascido em 2009 que reúne mantenedores de frameworks como Symfony, Laravel, Slim, Zend/Laminas, CakePHP, Drupal e outros. O objetivo declarado é simples: definir interfaces comuns para que componentes de diferentes ecossistemas possam ser combinados sem cola customizada. Antes das PSRs, trocar o logger do Symfony pelo do Zend exigia adapter manual; hoje, qualquer biblioteca que aceite `Psr\Log\LoggerInterface` aceita Monolog, KLogger, Symfony Logger ou qualquer outra implementação compatível.

Em APIs HTTP modernas, seis PSRs formam a espinha dorsal arquitetural. Cada uma resolve um ponto de acoplamento clássico:

- **PSR-7** (aprovada em 2015) — `RequestInterface`, `ResponseInterface`, `ServerRequestInterface`, `StreamInterface`, `UriInterface`. Modela mensagens HTTP como **objetos imutáveis**.
- **PSR-17** (2018) — Factories para criar mensagens PSR-7 sem acoplar a uma implementação concreta (Guzzle, Laminas Diactoros, Nyholm/PSR7).
- **PSR-15** (2018) — `MiddlewareInterface` e `RequestHandlerInterface`. Padrão pipeline para servidor HTTP.
- **PSR-3** (2012) — `LoggerInterface` com oito níveis (RFC 5424: emergency, alert, critical, error, warning, notice, info, debug).
- **PSR-11** (2017) — `ContainerInterface` com apenas `get($id)` e `has($id)`. Suficiente para localizar serviços; não impõe forma de configuração.
- **PSR-18** (2019) — `ClientInterface` para clientes HTTP outbound. Análogo ao PSR-7, porém para consumo de APIs externas.

Há outras (PSR-4 autoload, PSR-6/16 cache, PSR-12 estilo, PSR-14 eventos, PSR-20 clock, PSR-21 i18n), mas as seis acima são as que definem o esqueleto HTTP. Uma característica importante: PSRs são **interfaces, não implementações**. O FIG não publica código executável — publica contratos. Implementar e distribuir cabe ao ecossistema (Guzzle, Slim, Symfony HTTP Foundation Bridge, etc.).

A imutabilidade do PSR-7 merece destaque histórico. Foi controversa: muitos vinham do mundo Symfony Request mutável e estranharam ter de escrever `$response = $response->withStatus(404)`. A justificativa é segurança em pipelines paralelos e previsibilidade: nenhum middleware pode alterar o objeto que outro middleware ainda detém. O padrão **with*** retorna sempre uma nova instância com cópia rasa dos atributos modificados — análogo a `String.replace` em Java ou tipos record em C#.

## Por que importa

O ganho concreto é **substituibilidade**: a mesma camada de aplicação roda em servidores HTTP completamente diferentes (Swoole, RoadRunner, Apache+mod_php, FrankenPHP, ReactPHP) trocando apenas o adapter de entrada. Times que precisam migrar de cgi/mod_php para PHP de longa vida (RoadRunner para reduzir cold start) não reescrevem controllers — apenas o bootstrap.

Outro ganho é **testabilidade**. Para testar um middleware PSR-15, basta instanciar um `ServerRequestInterface` em memória e um `RequestHandlerInterface` fake. Não há TestCase de framework, não há rodar servidor, não há HTTP de verdade. Testes ficam abaixo de 1ms por caso.

Em projetos de plataforma (multi-tenant, white label), PSRs também viabilizam **plugins**: clientes do produto entregam um pacote Composer que registra middleware no pipeline, sem patch no core. Sem padrão comum, cada plugin reinventaria o request/response.

Por fim, há o argumento de **carreira/contratação**: dev que conhece PSR-7/15 produz em qualquer codebase moderno PHP. Sem isso, fica preso ao framework específico.

## Exemplo de código PHP

Exemplo 1 — middleware de logging estruturado com correlação por trace-id:

```php
<?php
declare(strict_types=1);

use Psr\Http\Message\{ServerRequestInterface, ResponseInterface};
use Psr\Http\Server\{MiddlewareInterface, RequestHandlerInterface};
use Psr\Log\LoggerInterface;

final readonly class RequestLogger implements MiddlewareInterface
{
    public function __construct(private LoggerInterface $logger) {}

    public function process(
        ServerRequestInterface $req,
        RequestHandlerInterface $next,
    ): ResponseInterface {
        $start   = hrtime(true);
        $traceId = $req->getHeaderLine('X-Trace-Id') ?: bin2hex(random_bytes(8));

        $this->logger->info('http.request', [
            'method'   => $req->getMethod(),
            'path'     => $req->getUri()->getPath(),
            'trace_id' => $traceId,
        ]);

        // imutabilidade: withAttribute retorna NOVA request
        $res = $next->handle($req->withAttribute('trace_id', $traceId));
        $ms  = (hrtime(true) - $start) / 1e6;

        $this->logger->info('http.response', [
            'status'      => $res->getStatusCode(),
            'duration_ms' => $ms,
            'trace_id'    => $traceId,
        ]);

        return $res->withHeader('X-Trace-Id', $traceId);
    }
}
```

Exemplo 2 — pipeline PSR-15 puro, sem framework:

```php
<?php
declare(strict_types=1);

use Psr\Http\Message\{ResponseInterface, ServerRequestInterface};
use Psr\Http\Server\{MiddlewareInterface, RequestHandlerInterface};

final class Pipeline implements RequestHandlerInterface
{
    /** @param list<MiddlewareInterface> $middlewares */
    public function __construct(
        private array $middlewares,
        private RequestHandlerInterface $fallback,
    ) {}

    public function handle(ServerRequestInterface $req): ResponseInterface
    {
        $mw = array_shift($this->middlewares);
        if ($mw === null) {
            return $this->fallback->handle($req);
        }
        return $mw->process($req, $this);
    }
}
```

Exemplo 3 — uso de PSR-17 factories para desacoplar a criação de respostas:

```php
<?php
declare(strict_types=1);

use Psr\Http\Message\{ResponseFactoryInterface, StreamFactoryInterface};

final readonly class JsonResponder
{
    public function __construct(
        private ResponseFactoryInterface $responses,
        private StreamFactoryInterface $streams,
    ) {}

    public function ok(array $payload, int $status = 200): ResponseInterface
    {
        $body = $this->streams->createStream(
            json_encode($payload, JSON_THROW_ON_ERROR | JSON_UNESCAPED_UNICODE),
        );
        return $this->responses
            ->createResponse($status)
            ->withHeader('Content-Type', 'application/json; charset=utf-8')
            ->withBody($body);
    }
}
```

Exemplo 4 — cliente PSR-18 com retry e timeout, dependendo apenas das interfaces:

```php
<?php
declare(strict_types=1);

use Psr\Http\Client\{ClientInterface, ClientExceptionInterface};
use Psr\Http\Message\{RequestInterface, ResponseInterface};

final readonly class RetryingClient implements ClientInterface
{
    public function __construct(
        private ClientInterface $inner,
        private int $maxAttempts = 3,
    ) {}

    public function sendRequest(RequestInterface $request): ResponseInterface
    {
        $attempt = 0;
        $lastErr = null;
        while ($attempt < $this->maxAttempts) {
            try {
                $res = $this->inner->sendRequest($request);
                if ($res->getStatusCode() < 500) {
                    return $res;
                }
            } catch (ClientExceptionInterface $e) {
                $lastErr = $e;
            }
            $attempt++;
            usleep(100_000 * (2 ** $attempt)); // backoff exponencial
        }
        throw $lastErr ?? new \RuntimeException('exceeded retries');
    }
}
```

## Quando usar / quando não usar

- **PSR-7/15** — sempre em APIs novas. Mesmo que use Laravel/Symfony nativamente, exponha boundaries via PSR-7 para libs reutilizáveis.
- **PSR-3** — sempre. É praticamente custo zero e o ecossistema (Monolog, Sentry, Loki adapters) já assume isso.
- **PSR-11** — sempre que código for distribuído como lib. Não force PHP-DI ou Symfony DI no consumidor.
- **PSR-18** — quando consumir HTTP externo de forma testável. Mock do `ClientInterface` é trivial.
- **PSR-17** — quase obrigatório em libs que produzem responses; sem ela, lib trava qual implementação PSR-7 o consumidor usa.
- **Não exija** PSRs em scripts internos curtos (worker minúsculo, cron de uma linha). O overhead de container, factory e interface não compensa.
- **Não use** PSR-7 como modelo de domínio. Ela é mensagem HTTP, não DTO de aplicação.

## Armadilhas comuns

- **Esquecer imutabilidade**: `$res->withStatus(404)` sem reatribuir retorna `$res` original. Bug clássico que passa em code review distraído. Em PHP 8.4, encadeie sempre: `return $res->withStatus(404)->withHeader(...)`.
- **Stream consumida duas vezes**: `(string) $req->getBody()` move o ponteiro. Segunda leitura volta vazia. Use `$body->rewind()` antes ou faça `$body->getContents()` uma única vez e guarde.
- **Misturar PSR-7 com Symfony Request no mesmo controller**: escolha um boundary. Symfony tem bridge (`PsrHttpFactory`) que converte; use em uma camada só.
- **Logger PSR-3 sem placeholders**: `$logger->info("user $id logged in")` interpola eagerly e impede log estruturado. Correto: `$logger->info('user {id} logged in', ['id' => $id])` — backend pode emitir JSON com campo `id` separado.
- **Container PSR-11 como service locator**: injetar `ContainerInterface` em domínio e fazer `$container->get(Foo::class)` espalha conhecimento do container. Use só na composition root.
- **PSR-15 com middlewares com estado mutável**: middlewares devem ser stateless (entre requests). Em workers de longa vida (RoadRunner/Swoole) qualquer `$this->lastUser` vaza dados entre requisições.
- **Reimplementar PSR-7**: tentar criar a "própria Request mais simples". Já existe Nyholm/PSR7 (rápido, mínimo) e Laminas Diactoros. Não há razão para reinvenção.
- **Não validar `Content-Length` em stream**: ler corpo grande sem limite leva a OOM. Use `LimitStream` ou checar `getSize()` antes.

## Links relacionados

- [[REST-Maduro]]
- [[Versionamento-Idempotencia]]
- [[OpenAPI]]
- [[JWT-vs-Sessoes]]
- [[Infra/Observabilidade-OWASP]]
- [[Performance/Cache-PSR-6-16]]

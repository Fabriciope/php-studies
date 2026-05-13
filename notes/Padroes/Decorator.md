---
tags: [php, padroes, decorator]
fase: 3
status: draft
---

# Decorator

## Conceito

Decorator é um padrão estrutural do GoF: **embrulhar um objeto em outro que implementa a mesma interface, adicionando comportamento antes/depois da chamada interna**. A unidade externa "decora" a interna; do ponto de vista do caller, ambos são intercambiáveis (mesma interface). É a expressão canônica do princípio **"composição sobre herança"**.

```
         interface OrderRepository
                  ^
                  |
   +--------------+-------------+--------------+
   |              |             |              |
PdoRepo    LoggingRepo    CachedRepo    MetricsRepo
                  |             |              |
                  +-------------+--------------+
                                | (wrap)
                                v
                              inner

  Composição típica:
    $repo = new MetricsRepo(
              new CachedRepo(
                new LoggingRepo(
                  new PdoRepo($pdo)
                )
              )
            );
```

Distinções:

| Padrão | Foco | Diferença chave |
|---|---|---|
| **Decorator** | Adicionar comportamento mantendo a mesma interface | Composição transparente; caller não percebe |
| **Proxy** | Controlar acesso a um recurso (lazy, remoto, segurança) | Mesma interface, mas intenção é **controle**, não acréscimo de feature |
| **Adapter** | Traduzir interfaces incompatíveis | Muda a interface; Decorator preserva |
| **Chain of Responsibility** | Lista de handlers que decidem se tratam | Cada handler pode parar a cadeia; Decorator sempre repassa |
| **Middleware (PSR-15)** | Decorator especializado em request/response HTTP | Mesma estrutura: empilhamento, repasse, intercepção |

Decorators são **stackable**: você compõe pilhas funcionais por configuração (cache → logger → metrics → adapter base). Cada camada cuida de **um concern ortogonal**. Adicionar uma nova feature (rate limit, retry, circuit breaker) = nova classe, sem tocar nas demais.

Em PSR-15 (HTTP middlewares), o padrão é literalmente Decorator: cada middleware implementa `MiddlewareInterface` e devolve uma `ResponseInterface` chamando o próximo. PHP comunidade usa também para CommandBus, QueryBus, HTTP Client (PSR-18 + plugins).

## Por que importa

Sem Decorator, concerns ortogonais (cache, log, métrica, retry, autorização) infiltram-se na classe de domínio. A classe `PdoOrderRepository` ganha código de cache, depois código de log, depois métricas — vira poliestrutura, violando SRP, dificultando teste e impedindo composição flexível.

Com Decorator:

- **Concerns isolados**: cada decorator existe em uma classe pequena, testável.
- **Composição por configuração**: produção tem `Metrics(Cache(Pdo))`; teste tem só `Pdo` ou `InMemory`.
- **Adição sem modificação**: novo concern = nova classe, intoque o resto (OCP).
- **Reuso entre serviços**: um `LoggingDecorator` genérico pode ser aplicado a vários repositories diferentes (com tipos).

Cenários concretos:

- **Cache de leitura** sobre repository.
- **Retry exponencial** sobre HTTP client.
- **Rate limiting** sobre gateway de pagamento.
- **Métricas/observabilidade** sobre command bus.
- **Autorização** sobre query bus.
- **Logging estruturado** sobre qualquer porta de saída.

## Exemplo de código PHP

### 1) Decorator de cache sobre Repository

```php
<?php
declare(strict_types=1);

use Psr\SimpleCache\CacheInterface;

final readonly class CachedOrderRepository implements OrderRepository
{
    public function __construct(
        private OrderRepository $inner,
        private CacheInterface $cache,
        private int $ttl = 300,
    ) {}

    public function ofId(OrderId $id): ?Order
    {
        $key = "order:{$id->toString()}";
        $hit = $this->cache->get($key);
        if ($hit instanceof Order) {
            return $hit;
        }

        $order = $this->inner->ofId($id);
        if ($order !== null) {
            $this->cache->set($key, $order, $this->ttl);
        }
        return $order;
    }

    public function save(Order $order): void
    {
        $this->inner->save($order);
        // Write-through: invalidar para forçar re-leitura coerente.
        $this->cache->delete("order:{$order->id->toString()}");
    }

    public function pendingOlderThan(\DateTimeImmutable $cutoff): iterable
    {
        // Não cacheia listas (volatilidade alta); apenas repassa.
        return $this->inner->pendingOlderThan($cutoff);
    }

    public function delete(OrderId $id): void
    {
        $this->inner->delete($id);
        $this->cache->delete("order:{$id->toString()}");
    }
}
```

### 2) Decorator de logging

```php
<?php
declare(strict_types=1);

use Psr\Log\LoggerInterface;

final readonly class LoggingOrderRepository implements OrderRepository
{
    public function __construct(
        private OrderRepository $inner,
        private LoggerInterface $logger,
    ) {}

    public function ofId(OrderId $id): ?Order
    {
        $start = hrtime(true);
        try {
            $order = $this->inner->ofId($id);
            $this->logger->debug('order.ofId', [
                'id' => $id->toString(),
                'hit' => $order !== null,
                'ms' => (hrtime(true) - $start) / 1e6,
            ]);
            return $order;
        } catch (\Throwable $e) {
            // NUNCA engolir: relançar.
            $this->logger->error('order.ofId.failed', [
                'id' => $id->toString(),
                'error' => $e->getMessage(),
            ]);
            throw $e;
        }
    }

    public function save(Order $order): void
    {
        $this->logger->info('order.save', ['id' => $order->id->toString()]);
        $this->inner->save($order);
    }

    public function pendingOlderThan(\DateTimeImmutable $cutoff): iterable
    {
        return $this->inner->pendingOlderThan($cutoff);
    }

    public function delete(OrderId $id): void
    {
        $this->inner->delete($id);
    }
}
```

### 3) Decorator de métricas

```php
<?php
declare(strict_types=1);

final readonly class MetricsOrderRepository implements OrderRepository
{
    public function __construct(
        private OrderRepository $inner,
        private MetricsCollector $metrics,
    ) {}

    public function ofId(OrderId $id): ?Order
    {
        $start = hrtime(true);
        $status = 'ok';
        try {
            return $this->inner->ofId($id);
        } catch (\Throwable $e) {
            $status = 'error';
            throw $e;
        } finally {
            // Meça também falhas: latência e taxa de erro são igualmente importantes.
            $this->metrics->observe('repo.ofId', (hrtime(true) - $start) / 1e6, [
                'status' => $status,
            ]);
        }
    }

    public function save(Order $order): void { $this->inner->save($order); }
    public function pendingOlderThan(\DateTimeImmutable $cutoff): iterable { return $this->inner->pendingOlderThan($cutoff); }
    public function delete(OrderId $id): void { $this->inner->delete($id); }
}
```

### 4) Composição declarativa

```php
<?php
declare(strict_types=1);

// Construa a pilha no wiring (container). Ordem importa:
//   - Logs envolvem tudo (vê inclusive cache hit).
//   - Metrics no meio.
//   - Cache mais perto do disco.

$base = new PdoOrderRepository($pdo, $hydrator);
$repo = new LoggingOrderRepository(
    new MetricsOrderRepository(
        new CachedOrderRepository($base, $cache, ttl: 300),
        $metrics,
    ),
    $logger,
);

// Em testes:
$repo = new InMemoryOrderRepository();
```

### 5) PSR-15 Middleware (Decorator no mundo HTTP)

```php
<?php
declare(strict_types=1);

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

// Cada middleware é um Decorator sobre o handler interno.
final readonly class RateLimitMiddleware implements MiddlewareInterface
{
    public function __construct(private RateLimiter $limiter) {}

    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler,
    ): ResponseInterface {
        $ip = $request->getServerParams()['REMOTE_ADDR'] ?? 'unknown';
        if (!$this->limiter->allow($ip)) {
            return new Response(status: 429, body: 'Too Many Requests');
        }
        return $handler->handle($request);
    }
}
```

## Quando usar / quando não usar

- **Usar** para cache, log, retry, métricas, autorização e auditoria sobre porta de saída — caso clássico de retorno alto.
- **Usar** quando ≥2 concerns ortogonais se acumulariam na mesma classe.
- **Usar** quando a composição varia por ambiente (dev/staging/prod com diferentes níveis de log e cache).
- **Usar** middlewares HTTP (PSR-15), command buses e query buses — Decorator é a estrutura natural.
- **Não usar** quando o decorator precisa conhecer o **tipo concreto** interno (`if ($this->inner instanceof PdoRepo)`) — virou herança disfarçada e quebra substituibilidade.
- **Cuidado com profundidade**: 6 decorators empilhados viram pesadelo de debug (stack trace gigante, ordem confusa). Limite a 3-4 camadas e considere [[Pipeline]] para fluxos lineares com semântica de "etapa".

## Armadilhas comuns

- **Esquecer de invalidar cache em `save`**: clássico de cache write-through mal feito. Leitura subsequente devolve dado obsoleto. Decorator de cache **tem** que invalidar nas operações de escrita.
- **Decorator de log engolindo exceção**: catch que loga e não relança transforma erro em silêncio. Crime. Sempre relance.
- **Métricas só de sucesso**: medir latência só quando dá certo distorce dashboards. Meça também latência de falha e taxa de erro — `try/finally` é seu amigo.
- **Ordem da pilha sem critério**: log fora ou dentro do cache muda o que se vê (cache hit é logado? deveria). Defina ordem por significado: o que está **fora** vê tudo; o que está **dentro** vê só I/O real.
- **Decorator com estado mutável**: contador, flag interna. Quebra idempotência e dificulta concorrência. Mantenha decorators `readonly`.
- **Re-implementar interface inteira só pra passar**: todos os métodos delegam — sem ganho. Use Decorator só onde adiciona valor; se quase tudo é passthrough, talvez sobre design.
- **Vazar `Throwable` do inner com tipo errado**: decorator que captura `Throwable` e relança como `\RuntimeException` perde stack trace original. Sempre passe `previous: $e`.
- **Decorator que muda contrato**: alterar retorno (ex.: `?Order` virar `Order|null|stdClass`) viola substituibilidade e quebra LSP. Mantenha o contrato.

## Links relacionados

- [[Repository]]
- [[Strategy]]
- [[Factory]]
- [[Observer]]
- [[Specification]]
- [[Padroes/SOLID]]
- [[Performance/Cache-PSR-6-16]]

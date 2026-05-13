---
tags: [php, performance, cache, psr]
fase: 6
status: draft
---

# Cache: PSR-6 vs PSR-16

## Conceito

Cache é, em essência, uma troca consciente de **consistência por latência**: armazenar o resultado de um cálculo caro (query SQL, chamada HTTP, render de template, operação criptográfica) para reutilizá-lo enquanto for "válido o suficiente". O PHP-FIG normalizou duas interfaces para essa camada — **PSR-6** (CacheItemPool, 2015) e **PSR-16** (SimpleCache, 2017) — permitindo trocar backends (APCu local → Redis em cluster → Memcached) sem reescrever consumidores.

**PSR-6** é o padrão "verboso e completo". Trabalha com dois objetos: `CacheItemPoolInterface` (a piscina, equivalente ao backend) e `CacheItemInterface` (um item, com `isHit()`, `get()`, `set()`, `expiresAfter()`, `expiresAt()`). O fluxo é: pedir item ao pool, verificar se há hit, caso contrário calcular e popular, salvar de volta. A vantagem é controle granular: metadata por item, deferred saves (acumular `saveDeferred` + `commit` final para reduzir round-trips), tagging via extensões (`TagAwareAdapterInterface` na Symfony Cache, `TaggableCacheItemPoolInterface` no Cache PSR-6 ext). A desvantagem é boilerplate — cinco linhas para o que em PSR-16 são duas.

**PSR-16** (`Psr\SimpleCache\CacheInterface`) é o estilo "Map de Map": `get`, `set`, `delete`, `has`, `getMultiple`, `setMultiple`, `clear`. Mais limpo, mais próximo de quem vem de Memcached/Redis cru. Não suporta tags nativamente, não tem deferred saves. Para 90% dos casos é o que você quer.

**Backends comuns**:

| Backend     | Latência | Persistência | Distribuído | Uso típico                        |
| ----------- | -------- | ------------ | ----------- | --------------------------------- |
| APCu        | ~1µs     | não (RAM)    | não         | cache local por servidor, hot     |
| Memcached   | ~0.5ms   | não          | sim (LRU)   | cache compartilhado simples       |
| Redis       | ~0.5ms   | opcional     | sim         | cache + estruturas + pubsub       |
| Filesystem  | ~5-50ms  | sim          | NFS/EFS     | dev, cache muito grande           |
| Array (mem) | ~0.1µs   | só request   | não         | dedup dentro da request           |

**Cache stampede** (ou "dog-piling") é a falha clássica: uma chave popular expira, simultaneamente 1000 requests batem na cache vazia, todas vão à fonte (banco), banco trava, app cai. Mitigações: **lock** explícito (apenas 1 request reconstrói, outras esperam ou servem stale), **probabilistic early expiration** (XFetch: cada request decide aleatoriamente recalcular antes do TTL, com probabilidade crescente perto da expiração), **stale-while-revalidate** (servir valor velho enquanto recalcula em background).

**Estratégias de invalidação** definem como dados velhos saem do cache:

- **TTL puro** — expira por tempo; simples, mas dados ficam stale até expirar.
- **Write-through** — toda escrita na fonte propaga para cache; consistência forte, latência maior no write.
- **Write-around** — write vai direto pra fonte, cache invalida (`delete`); ok quando dados raramente são lidos imediatamente após escrita.
- **Write-back (write-behind)** — write rápido em cache, async para fonte; **perigoso** se cache cai (perde dados).
- **Cache-aside (lazy)** — aplicação consulta cache, falha → consulta fonte → popula cache; padrão dominante.
- **Tagging** — agrupa entradas por tag, invalida em massa (`invalidateTags(['user.42'])`); requer backend que suporte (Redis, filesystem com Symfony).

Distinção importante: **cache de leitura** (dados consultados, ex.: lista de produtos) versus **cache de resposta HTTP** (resposta completa, headers + body) versus **cache de fragmento** (parte de página, ex.: header logado). O último, em PHP, é geralmente feito por middleware PSR-15 ou por reverse proxy (Varnish, Nginx fastcgi_cache). PSR-6/16 cobre o primeiro caso primariamente.

## Por que importa

Performance, mas também resiliência. Em uma API que serve catálogo de produtos:

| Configuração                                  | RPS  | p95   | DB QPS |
| --------------------------------------------- | ---- | ----- | ------ |
| Sem cache                                     | 240  | 320ms | 1800   |
| PSR-16 + Redis (TTL 60s)                      | 1850 | 18ms  | 35     |
| PSR-16 + Redis + APCu L1 (TTL 5s/60s)         | 2400 | 9ms   | 35     |
| Idem + stale-while-revalidate                 | 2410 | 8ms   | 35     |

A introdução de **cache em duas camadas** (L1=APCu por servidor, L2=Redis compartilhado) reduz pressão na rede e elimina o "1000-thundering-herd" comum quando Redis precisa servir 50 servidores PHP. SWR remove o cliff de latência no momento da expiração.

Anti-cenário: durante Black Friday, time desabilitou cache "para garantir dados frescos". DB derrubou em 4 minutos. **Cache não é otimização opcional para sistemas de alta carga — é parte da arquitetura.**

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// === PSR-16: o caso comum ===
use Psr\SimpleCache\CacheInterface;

final readonly class ExchangeRateService
{
    public function __construct(
        private CacheInterface $cache,
        private ExchangeRateApi $api,
        private LoggerInterface $log,
    ) {}

    public function rate(string $from, string $to, \DateTimeImmutable $date): float
    {
        $key = sprintf('rate.%s.%s.%s', $from, $to, $date->format('Y-m-d'));

        $cached = $this->cache->get($key);
        if ($cached !== null) {
            return $cached;
        }

        $rate = $this->api->fetch($from, $to, $date);
        // taxas históricas são imutáveis: TTL alto
        $this->cache->set($key, $rate, ttl: 86400 * 7);
        return $rate;
    }
}
```

```php
<?php
declare(strict_types=1);

// === PSR-6: tags, deferred saves, metadata ===
use Psr\Cache\CacheItemPoolInterface;
use Symfony\Contracts\Cache\TagAwareCacheInterface;

final readonly class OrderCache
{
    public function __construct(private TagAwareCacheInterface $cache) {}

    public function get(OrderId $id, callable $loader): Order
    {
        return $this->cache->get(
            key: "order.{$id->toString()}",
            callback: function (\Symfony\Contracts\Cache\ItemInterface $item) use ($id, $loader) {
                $item->expiresAfter(300);
                $item->tag(["order", "customer.{$loader()->customerId}"]);
                return $loader();
            },
        );
    }

    /** Invalida TODOS os pedidos de um cliente em uma operação */
    public function invalidateCustomer(CustomerId $customerId): void
    {
        $this->cache->invalidateTags(["customer.{$customerId->toString()}"]);
    }
}
```

```php
<?php
declare(strict_types=1);

// === Cache stampede: lock + XFetch ===
final class StampedeProtectedCache
{
    public function __construct(
        private CacheInterface $cache,
        private \Redis $redis,        // para lock
        private float $beta = 1.0,    // agressividade do XFetch
    ) {}

    /**
     * XFetch: probabilistic early expiration.
     * Próximo da expiração, requests aleatoriamente decidem recalcular.
     */
    public function get(string $key, int $ttl, callable $compute): mixed
    {
        $payload = $this->cache->get($key);

        if ($payload !== null) {
            ['value' => $v, 'delta' => $delta, 'expiry' => $expiry] = $payload;
            $now = microtime(true);
            // XFetch formula: recompute if (now - delta * beta * log(rand)) >= expiry
            if ($now - $delta * $this->beta * log(mt_rand() / mt_getrandmax()) < $expiry) {
                return $v;
            }
        }

        // Lock para evitar stampede de cálculo
        $lockKey = "lock:$key";
        $haveLock = $this->redis->set($lockKey, '1', ['nx', 'ex' => 30]);

        if (!$haveLock && $payload !== null) {
            // Não conseguimos lock, mas temos valor velho: serve stale
            return $payload['value'];
        }

        try {
            $t0 = microtime(true);
            $value = $compute();
            $delta = microtime(true) - $t0; // tempo de cálculo (XFetch usa)

            $this->cache->set($key, [
                'value'  => $value,
                'delta'  => $delta,
                'expiry' => microtime(true) + $ttl,
            ], $ttl + 60); // TTL real maior que lógico, para servir stale

            return $value;
        } finally {
            if ($haveLock) {
                $this->redis->del($lockKey);
            }
        }
    }
}
```

```php
<?php
declare(strict_types=1);

// === Cache em duas camadas (L1 APCu + L2 Redis) ===
final class TieredCache implements CacheInterface
{
    public function __construct(
        private CacheInterface $l1,   // APCu — TTL curto (1-10s)
        private CacheInterface $l2,   // Redis — TTL longo
        private int $l1Ttl = 5,
    ) {}

    public function get(string $key, mixed $default = null): mixed
    {
        $value = $this->l1->get($key);
        if ($value !== null) {
            return $value; // hit L1: ~1µs
        }

        $value = $this->l2->get($key);
        if ($value !== null) {
            $this->l1->set($key, $value, $this->l1Ttl);
            return $value;
        }

        return $default;
    }

    public function set(string $key, mixed $value, int|\DateInterval|null $ttl = null): bool
    {
        $this->l1->set($key, $value, $this->l1Ttl);
        return $this->l2->set($key, $value, $ttl);
    }

    // delete, has, etc. — propagar para ambas as camadas
    public function delete(string $key): bool
    {
        $this->l1->delete($key);
        return $this->l2->delete($key);
    }

    // ... demais métodos da interface
}
```

```yaml
# Symfony Cache — configuração típica com Redis + tags
framework:
    cache:
        app: cache.adapter.redis_tag_aware
        default_redis_provider: '%env(REDIS_DSN)%'
        pools:
            cache.users:
                adapter: cache.adapter.redis_tag_aware
                default_lifetime: 3600
            cache.heavy_reports:
                adapter: cache.adapter.filesystem
                default_lifetime: 86400
```

## Quando usar / quando não usar

- **PSR-16** — default para 90% dos casos. Mais limpo, menos boilerplate.
- **PSR-6** — quando precisa de tags, deferred saves, ou lib obriga (Doctrine 2 metadata cache).
- **APCu** — cache **por servidor** (não compartilhado). Excelente para dados pequenos, lidos muitas vezes (configurações, feature flags, lookup tables). Em ambiente com 1 servidor: substitui Redis.
- **Redis** — cache compartilhado, estruturas (lists, sets, sorted sets), pubsub. Default para apps multi-servidor.
- **Memcached** — mais simples que Redis, sem persistência. Hoje raramente escolhido sobre Redis, exceto por inércia de stack legada.
- **Filesystem** — dev, ou cache muito grande (>10MB por item) que não cabe em Redis. Lento em NFS.
- **Cache de resposta HTTP** — quando a response é determinística para `(URL, headers de Vary)`. Faça em reverse proxy (Varnish/Nginx) **antes** de PHP — não cacheie em PHP o que cache HTTP resolve melhor.
- **Não cachear** — dados altamente personalizados consultados raramente (custo > ganho); dados que mudam por segundo (TTL baixo demais = pressão de invalidação maior que ganho).
- **Não use** PSR-16 quando precisa de operações multi-key transacionais — vá direto ao backend.

## Armadilhas comuns

1. **Cache stampede em chave popular.** Item expira, 1000 requests simultâneas recalculam. *Por quê*: TTL puro sem lock nem early expiration. Mitigue com `Symfony\Contracts\Cache\CacheInterface::get($key, $callback)` que já faz lock interno, ou implemente XFetch.

2. **Chave colidindo entre tenants.** `cache.get("user.42")` retorna user 42 do tenant A para tenant B. *Por quê*: ausência de prefixo de tenant na chave. **Regra**: chave de cache deve conter *todos* os eixos de variabilidade (tenant, locale, role).

3. **TTL longo sem invalidação no write.** Você atualiza um produto, mas o cache continua mostrando o preço antigo por 1h. *Por quê*: write-around não foi implementado — `productRepository->save()` precisa chamar `cache->delete("product.42")`.

4. **Cachear objeto não-serializável.** `cache->set('pdo', $pdoConnection)` falha silenciosamente ou estoura. *Por quê*: `serialize` de PDO, Redis client, resource — proibido. Cacheie apenas DTOs/valores primitivos.

5. **Backend remoto sem timeout.** Redis fica lento ou cai, sua aplicação trava 30s por request esperando. *Por quê*: cliente Redis com timeout default infinito. Sempre configure `connect_timeout=1s`, `read_timeout=500ms`, e tenha fallback (passar direto ao DB se cache der erro).

6. **Tags excessivas.** Cada item tem 50 tags. Operação de `invalidateTags` vira O(items × tags). *Por quê*: zelo demais; tagging é poderoso mas tem custo. Restrinja a 2-4 tags úteis por item.

7. **Cache de objeto vivo (Doctrine entity gerenciada).** Cacheia entity, próxima request faz `unserialize`, EntityManager não conhece o objeto — alterações não persistem. *Por quê*: cache deve guardar **dados**, não objetos de domínio amarrados a contexto. Cacheie arrays/DTOs.

8. **Esquecer de Vary em cache de resposta HTTP.** Cache mostra página em PT para usuário EN porque ignora `Accept-Language`. *Por quê*: chave de cache HTTP deve incluir headers relevantes (`Vary: Accept-Language, Cookie`).

## Links relacionados

- [[Padroes/Decorator]]
- [[Filas]]
- [[APIs/Versionamento-Idempotencia]]
- [[Profiling]]
- [[OPcache-JIT-Preloading]]
- [[FrankenPHP-RoadRunner-Swoole]]
- [[Fibers-N1]]

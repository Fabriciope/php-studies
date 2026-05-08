---
tags: [php, performance, cache, psr]
fase: 6
status: stub
---

# Cache: PSR-6 vs PSR-16

## Conceito

Dois padrões PHP-FIG para cache:

- **PSR-6** (`CacheItemPoolInterface`) — orientado a item: você pega um `CacheItem`, verifica `isHit()`, atribui valor, salva. Verboso, mas suporta meta (TTL, tags via extensões).
- **PSR-16** (`CacheInterface`) — simples: `get`, `set`, `delete`, `has`. Estilo "key-value store". Boilerplate mínimo.

Ambos têm `CacheItem`/`CacheInterface` em libs como Symfony Cache, que oferecem adapters: APCu, Redis, Memcached, filesystem, in-memory.

## Por que importa

Cache mal projetado é a fonte mais comum de bugs sutis ("por que está retornando o valor antigo?"). Padronizar via PSR torna a camada **trocável** (APCu local → Redis em cluster) sem reescrever consumidores.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

use Psr\SimpleCache\CacheInterface;

// PSR-16 — get/set simples
final readonly class CachedExchangeRate {
    public function __construct(
        private CacheInterface $cache,
        private ExchangeRateApi $api,
    ) {}

    public function brlToUsd(\DateTimeImmutable $date): float {
        $key = "rate:brl-usd:{$date->format('Y-m-d')}";
        $cached = $this->cache->get($key);
        if ($cached !== null) return $cached;

        $rate = $this->api->fetch('BRL', 'USD', $date);
        $this->cache->set($key, $rate, 86400);
        return $rate;
    }
}

// PSR-6 — quando precisa de item rico (ex.: tags)
use Psr\Cache\CacheItemPoolInterface;

final readonly class TaggedOrderCache {
    public function __construct(private CacheItemPoolInterface $pool) {}

    public function remember(OrderId $id, callable $loader): Order {
        $item = $this->pool->getItem("order.{$id->toString()}");
        if ($item->isHit()) return $item->get();
        $order = $loader();
        $item->set($order)->expiresAfter(300);
        $this->pool->save($item);
        return $order;
    }
}
```

## Estratégias clássicas

- **Cache-aside** (lazy): aplicação consulta cache, falha → consulta fonte → grava cache. Mais comum.
- **Write-through**: aplicação grava em cache E fonte. Garante consistência, custa latência.
- **Write-behind**: grava cache, fonte assíncrona. Risco se cache morrer.
- **TTL + invalidação** sob escrita: combine TTL curto com `delete` em `save()`.

## Quando usar / quando não usar

- **PSR-16** — 90% dos casos. Mais limpo.
- **PSR-6** — tagging, expiração customizada por item, libs que exigem.
- **Não cachear** dados que mudam por usuário e raramente são reusados (custo > ganho).
- **Cuidado com cache stampede**: 1000 requests batendo na cache vencida → 1000 vão pra fonte. Use `lock` ou `early refresh`.

## Armadilhas comuns

- Cachear objeto com referência circular ou recurso (PDO, stream) — falha de serialize.
- TTL longo demais sem invalidação no write — dados velhos visíveis indefinidamente.
- Chave de cache colidindo entre tenants — vazamento entre clientes.
- Cachear em request síncrona com Redis remoto sem timeout — quando Redis cair, sua app cai junto.

## Links relacionados

- [[Padroes/Decorator]]
- [[Filas]]
- [[APIs/Versionamento-Idempotencia]]

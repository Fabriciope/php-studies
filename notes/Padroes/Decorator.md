---
tags: [php, padroes, decorator]
fase: 3
status: stub
---

# Decorator

## Conceito

Embrulhar um objeto em outro que implementa a **mesma interface**, adicionando comportamento antes/depois da chamada interna. Composição em vez de herança. Permite empilhar capacidades (cache, log, retry, métricas) sem tocar a classe base.

## Por que importa

É a forma idiomática de adicionar concerns ortogonais (cache, log) sem inflar a classe de domínio. Em PHP, cobre 90% do que outros ecosistemas chamam de "middleware" para serviços não-HTTP.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// decorator de cache sobre Repository
final class CachedOrderRepository implements OrderRepository {
    public function __construct(
        private OrderRepository $inner,
        private CacheInterface $cache,
        private int $ttl = 300,
    ) {}

    public function ofId(OrderId $id): ?Order {
        $key = "order:{$id->toString()}";
        if ($hit = $this->cache->get($key)) {
            return $hit;
        }
        $order = $this->inner->ofId($id);
        if ($order) {
            $this->cache->set($key, $order, $this->ttl);
        }
        return $order;
    }

    public function save(Order $order): void {
        $this->inner->save($order);
        $this->cache->delete("order:{$order->id->toString()}");
    }

    public function pendingOlderThan(\DateTimeImmutable $cutoff): iterable {
        return $this->inner->pendingOlderThan($cutoff);  // não cacheia listas
    }
}

// composição: $repo = new CachedOrderRepository(new LoggingRepo(new PdoRepo($pdo)));
```

## Quando usar / quando não usar

- **Cache, log, retry, métricas, autorização** sobre porta de saída — clássico.
- **Quando ≥2 concerns ortogonais** se acumulariam na mesma classe — extraia cada um para um decorator.
- **Não usar** se o decorator precisa conhecer o tipo concreto interno — virou herança disfarçada.
- **Cuidado com profundidade:** 6 decorators empilhados viram pesadelo de debug. Limite a 2–3 e considere [[Pipeline]].

## Armadilhas comuns

- Esquecer de invalidar cache no `save` — bug clássico de cache write-through mal feito.
- Decorator de log que **engole** exceção do inner para não "poluir" o log — crime.
- Métricas medindo só sucesso. Meça também latência de falha e taxa de erro.

## Links relacionados

- [[Repository]]
- [[Strategy]]
- [[Performance/Cache-PSR-6-16]]

---
tags: [php, performance, fibers, async, n1]
fase: 6
status: stub
---

# Fibers + diagnóstico de N+1

## Conceito

**Fibers** (PHP 8.1+) são unidades de execução cooperativas: o código pode `Fiber::suspend()` e ser retomado depois, em modelo single-thread. Não são threads — não rodam em paralelo. São a base para libs async (Amp, ReactPHP via fibers).

**N+1** é o anti-pattern em que listar N itens dispara 1 query inicial + N queries adicionais (uma por item). Em listas de 100 itens, 101 queries onde 2 bastariam.

(Os dois temas juntos porque ambos são sobre **evitar tempo de espera ocioso** — fibers cooperam para sobrepor I/O; eliminar N+1 elimina I/O redundante.)

## Por que importa

Fibers permitem clientes HTTP, banco e fila genuinamente concorrentes em PHP — algo que antes exigia Swoole ou multi-processo. N+1, por sua vez, é provavelmente **a causa #1 de slowness** em apps com ORM. Saber detectar e corrigir é skill básico sênior.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// === Fibers — composição de I/O ===
function httpGet(string $url): Fiber {
    return new Fiber(function () use ($url): string {
        $ch = curl_init($url);
        curl_setopt_array($ch, [CURLOPT_RETURNTRANSFER => true, CURLOPT_NOSIGNAL => 1]);
        // em código real: integrar com event loop (Amp/Revolt)
        Fiber::suspend();
        $body = curl_exec($ch);
        curl_close($ch);
        return $body;
    });
}

// uso conceitual: scheduler retoma quando socket fica pronto
$fa = httpGet('https://api.a.com')->start();
$fb = httpGet('https://api.b.com')->start();
// ambos em "espera" — scheduler resume quando há resposta

// === N+1 — diagnóstico ===
final class QueryLogger {
    public array $queries = [];
    public function log(string $sql, array $params): void {
        $this->queries[] = compact('sql', 'params');
    }
    public function detectN1(): array {
        $counts = [];
        foreach ($this->queries as $q) {
            $template = preg_replace('/= \?/', '= ?', $q['sql']);
            $counts[$template] = ($counts[$template] ?? 0) + 1;
        }
        return array_filter($counts, fn ($n) => $n > 5);
    }
}

// === N+1 — correção via "load and pluck" ===
$orders = $orderRepo->recent(50);
$customerIds = array_unique(array_map(fn ($o) => $o->customerId->toString(), $orders));
$customers = $customerRepo->byIds($customerIds);  // 1 query
$indexed = array_column($customers, null, 'id');
foreach ($orders as $o) $o->customer = $indexed[$o->customerId->toString()];
```

## Estratégias para eliminar N+1

- **Eager load**: SQL `JOIN` ou `WHERE id IN (...)` em uma query.
- **Identity map** + bulk loader.
- **DataLoader pattern** (do mundo GraphQL): bufferiza pedidos, dispara em lote.
- **Read model dedicado** (CQRS): query única denormalizada.

## Quando usar / quando não usar

- **Fibers** — quando você precisa de concorrência I/O em runtime persistente. Em FPM, costumam ser desnecessários (cada request já é isolada).
- **Não usar Fibers** para CPU-bound — é cooperativo, não paralelo.
- **Sempre** monitorar N+1 com query logger em dev/staging.

## Armadilhas comuns

- Fiber sem event loop — `suspend` sem ninguém para resumir = deadlock.
- Confundir Fibers com Threads.
- Detectar N+1 só em prod: prejudica usuário. Use linter/log em dev.
- Eager load **excessivo** (carregar 15 relations sempre) — vira "1 query gigante" igualmente lenta. Carregue só o necessário.

## Links relacionados

- [[FrankenPHP-RoadRunner-Swoole]]
- [[Profiling]]
- [[Padroes/Repository]]

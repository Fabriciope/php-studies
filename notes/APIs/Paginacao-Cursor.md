---
tags: [php, api, paginacao, performance]
fase: 5
status: stub
---

# Paginação Cursor

## Conceito

Em vez de `?page=42&size=20` (que vira `OFFSET 820` no banco), usa-se um **cursor opaco** que aponta para "depois deste registro": `?after=eyJpZCI6MTIzfQ&limit=20`. O servidor decodifica o cursor (geralmente JSON base64) e aplica `WHERE (created_at, id) > (?, ?)`.

## Por que importa

Offset escala mal: para chegar à página 1000, o banco lê 20.000 linhas e descarta 19.980. Com cursor, sempre lê 20. Em listagens grandes, é a diferença entre p95 de 50ms e p95 de 8s.

Bônus: **estabilidade** — em offset, se alguém insere registro na página 3 enquanto você pagina, vê duplicatas/lacunas. Cursor sobre `(timestamp, id)` é estável.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

final readonly class Cursor {
    public function __construct(
        public \DateTimeImmutable $createdAt,
        public int $id,
    ) {}

    public static function decode(string $opaque): ?self {
        $raw = base64_decode($opaque, true);
        if ($raw === false) return null;
        $data = json_decode($raw, true);
        if (!is_array($data) || !isset($data['t'], $data['i'])) return null;
        return new self(new \DateTimeImmutable($data['t']), (int) $data['i']);
    }

    public function encode(): string {
        return base64_encode(json_encode([
            't' => $this->createdAt->format(\DateTimeInterface::ATOM),
            'i' => $this->id,
        ]));
    }
}

final readonly class OrderListQuery {
    public function __construct(private \PDO $pdo) {}

    public function list(?Cursor $after, int $limit = 20): array {
        $limit = min($limit, 100);
        $sql = 'SELECT id, created_at, total_cents FROM orders';
        $params = [];
        if ($after) {
            $sql .= ' WHERE (created_at, id) > (?, ?)';
            $params = [$after->createdAt->format('Y-m-d H:i:s'), $after->id];
        }
        $sql .= ' ORDER BY created_at, id LIMIT ' . ($limit + 1);

        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($params);
        $rows = $stmt->fetchAll(\PDO::FETCH_ASSOC);

        $hasMore = count($rows) > $limit;
        $rows = array_slice($rows, 0, $limit);

        $next = $hasMore && $rows
            ? new Cursor(new \DateTimeImmutable(end($rows)['created_at']), (int) end($rows)['id'])
            : null;

        return [
            'data' => $rows,
            'next_cursor' => $next?->encode(),
        ];
    }
}
```

## Quando usar / quando não usar

- **Sempre que a lista cresce** (pedidos, eventos, mensagens).
- **Não usar cursor** em listas pequenas e estáveis (estados do Brasil) — offset é mais simples.
- **Bidirecional** se UI precisa "voltar uma página" — adicione `before` simétrico.

## Armadilhas comuns

- Cursor sobre coluna **não-única** (`created_at` apenas) — empate causa loops/skips. Sempre use **par** com PK.
- Index ausente sobre `(created_at, id)` — query lenta apesar do cursor.
- Expor cursor não opaco (id puro) — cliente fica acoplado ao schema interno.
- Esquecer `LIMIT n+1` truque para detectar `has_more` sem `COUNT(*)`.

## Links relacionados

- [[REST-Maduro]]
- [[Performance/Fibers-N1]]
- [[OpenAPI]]

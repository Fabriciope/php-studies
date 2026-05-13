---
tags: [php, api, paginacao, performance]
fase: 5
status: draft
---

# Paginação Cursor

## Conceito

Paginação é o mecanismo de devolver listas grandes em pedaços. Dois modelos dominam o ecossistema HTTP:

- **Offset-based** (clássico): `GET /orders?page=42&size=20`. O servidor traduz para `SELECT ... ORDER BY created_at LIMIT 20 OFFSET 820`.
- **Cursor-based** (também chamado *keyset* ou *seek* pagination): `GET /orders?after=eyJpZCI6MTIzfQ&limit=20`. O servidor decodifica um marcador opaco e aplica `WHERE (created_at, id) > (?, ?) ORDER BY created_at, id LIMIT 20`.

A diferença parece sutil, mas a complexidade no banco é radicalmente distinta. Em offset, para chegar à página N o banco precisa **ler e descartar** N*size linhas — é O(N) por página. Com índice apropriado em `(created_at, id)`, o cursor faz um **seek** direto na árvore B-tree para o primeiro registro maior que o último visto: O(log n) constante, independente da posição.

A técnica é antiga (descrita em "Pagination Done the PostgreSQL Way" de Markus Winand, 2013) e foi popularizada nas APIs de Twitter (`max_id`/`since_id`), Facebook Graph (`cursors.after`/`before`), Stripe (`starting_after`/`ending_before`) e Shopify. GraphQL formalizou no padrão **Connections/Relay** (`edges`, `pageInfo`, `endCursor`, `hasNextPage`).

Cursores podem ser:

- **Opacos** — string sem significado público (base64 de JSON, ou um UUID em tabela de sessões de paginação). Servidor pode mudar o formato sem quebrar cliente.
- **Transparentes** — id puro ou timestamp visível. Mais simples, mais frágil (acopla cliente ao schema).
- **Estáveis** — baseados em chave única ordenada (`(created_at, id)`). Sempre apontam para o mesmo registro mesmo com inserções concorrentes.
- **Instáveis** — baseados só em offset numérico ou em coluna não-única. Sofrem com concorrência.

O **encoding** mais comum em REST é `base64url(json_encode(['t' => ..., 'i' => ...]))`. Assinar (HMAC) opcionalmente impede que cliente manipule o cursor para injetar SQL ou pular registros não autorizados.

Variantes importantes:

- **Bidirecional**: além de `after`, expõe `before` para "página anterior". Requer ordenação reversa e cuidados com inclusividade.
- **Cursor + total count caro**: às vezes UI precisa de "página 3 de 42". Total exato custa `COUNT(*)`. Pratique-se uma estimativa (`pg_class.reltuples` no Postgres) ou ofereça "mais de 10.000" em vez de número exato.
- **Stable cursor com soft-delete**: se um registro do meio é soft-deletado, o cursor ainda funciona — apenas não retorna aquele item.

## Por que importa

Offset escala mal e silenciosamente. Em desenvolvimento (dataset de 1k linhas), página 50 é tão rápida quanto página 1. Em produção (dataset de 5M linhas), página 1 leva 30ms e página 5000 leva 8 segundos. Pior: o time só descobre quando um cliente real (admin filtrando histórico, scraping de parceiro) navega fundo.

Cenário concreto 1 — **admin paginando 100k pedidos**: com offset, página 4000 lê 80k linhas, faz `ORDER BY` parcial e descarta 79.980. p95 vai a 12s, banco trava conexões. Com cursor, sempre lê 20. p95 = 25ms.

Cenário concreto 2 — **scroll infinito em feed social**: enquanto usuário rola, novos posts são criados no topo. Em offset, ao buscar página 2 ele vê post duplicado (que era da página 1 antes da inserção). Em cursor baseado em `id`, isso não acontece — o cursor aponta para um id específico, não posição.

Cenário concreto 3 — **export CSV de 2M linhas**: com offset, o tempo total cresce O(n²). Com cursor, cresce O(n). Diferença entre rodar em 5 minutos vs 5 horas.

Cenário concreto 4 — **arquitetura sharded**: offset global em dados shardados exige `LIMIT/OFFSET` em cada shard e merge no app. Cursor por chave de shard simplifica drasticamente.

## Exemplo de código PHP

Exemplo 1 — cursor estruturado:

```php
<?php
declare(strict_types=1);

final readonly class Cursor
{
    public function __construct(
        public \DateTimeImmutable $createdAt,
        public int $id,
    ) {}

    public static function decode(string $opaque): ?self
    {
        $raw = base64_decode(strtr($opaque, '-_', '+/'), true);
        if ($raw === false) {
            return null;
        }
        $data = json_decode($raw, true);
        if (!is_array($data) || !isset($data['t'], $data['i'])) {
            return null;
        }
        try {
            return new self(new \DateTimeImmutable($data['t']), (int) $data['i']);
        } catch (\Exception) {
            return null;
        }
    }

    public function encode(): string
    {
        $json = json_encode([
            't' => $this->createdAt->format(\DateTimeInterface::ATOM),
            'i' => $this->id,
        ], JSON_THROW_ON_ERROR);
        return rtrim(strtr(base64_encode($json), '+/', '-_'), '=');
    }
}
```

Exemplo 2 — query keyset com truque `LIMIT n+1` para detectar há mais:

```php
<?php
declare(strict_types=1);

final readonly class OrderListQuery
{
    public function __construct(private \PDO $pdo) {}

    /**
     * @return array{data: list<array>, next_cursor: ?string, has_more: bool}
     */
    public function list(?Cursor $after, int $limit = 20): array
    {
        $limit = max(1, min($limit, 100)); // clamp para evitar abuse

        $sql = 'SELECT id, created_at, total_cents
                FROM orders';
        $params = [];

        if ($after !== null) {
            // row-value comparison: precisa Postgres ou MySQL 8+
            $sql .= ' WHERE (created_at, id) > (?, ?)';
            $params[] = $after->createdAt->format('Y-m-d H:i:s.u');
            $params[] = $after->id;
        }
        // ORDER BY DEVE ser igual ao usado no WHERE — index match
        $sql .= ' ORDER BY created_at, id LIMIT ' . ($limit + 1);

        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($params);
        $rows = $stmt->fetchAll(\PDO::FETCH_ASSOC);

        $hasMore = count($rows) > $limit;
        if ($hasMore) {
            array_pop($rows);
        }

        $next = ($hasMore && $rows !== [])
            ? new Cursor(
                new \DateTimeImmutable(end($rows)['created_at']),
                (int) end($rows)['id'],
              )
            : null;

        return [
            'data'        => $rows,
            'next_cursor' => $next?->encode(),
            'has_more'    => $hasMore,
        ];
    }
}
```

Exemplo 3 — cursor bidirecional (suporte a `before`):

```php
<?php
declare(strict_types=1);

enum Direction
{
    case After;
    case Before;
}

final readonly class BidirectionalList
{
    public function __construct(private \PDO $pdo) {}

    public function list(?Cursor $cursor, Direction $dir, int $limit = 20): array
    {
        $op    = $dir === Direction::After ? '>' : '<';
        $order = $dir === Direction::After ? 'ASC' : 'DESC';

        $sql = "SELECT id, created_at, total_cents FROM orders";
        $params = [];

        if ($cursor !== null) {
            $sql .= " WHERE (created_at, id) {$op} (?, ?)";
            $params = [$cursor->createdAt->format('c'), $cursor->id];
        }
        $sql .= " ORDER BY created_at {$order}, id {$order} LIMIT " . ($limit + 1);

        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($params);
        $rows = $stmt->fetchAll(\PDO::FETCH_ASSOC);

        // se foi Before, reverte para manter ordem cronológica na resposta
        if ($dir === Direction::Before) {
            $rows = array_reverse($rows);
        }
        return $rows;
    }
}
```

Exemplo 4 — cursor assinado (anti-tampering com HMAC):

```php
<?php
declare(strict_types=1);

final readonly class SignedCursor
{
    public function __construct(private string $secret) {}

    public function encode(array $payload): string
    {
        $json = json_encode($payload, JSON_THROW_ON_ERROR);
        $sig  = substr(hash_hmac('sha256', $json, $this->secret), 0, 16);
        return rtrim(strtr(base64_encode($json . '.' . $sig), '+/', '-_'), '=');
    }

    public function decode(string $token): ?array
    {
        $raw = base64_decode(strtr($token, '-_', '+/'), true);
        if ($raw === false) {
            return null;
        }
        $dot = strrpos($raw, '.');
        if ($dot === false) {
            return null;
        }
        $json = substr($raw, 0, $dot);
        $sig  = substr($raw, $dot + 1);
        $exp  = substr(hash_hmac('sha256', $json, $this->secret), 0, 16);
        if (!hash_equals($exp, $sig)) {
            return null;
        }
        return json_decode($json, true);
    }
}
```

## Quando usar / quando não usar

- **Sempre cursor** em listas que crescem sem teto: pedidos, eventos, logs, mensagens, transações.
- **Cursor** quando a UI faz scroll infinito ou polling para detectar novos itens.
- **Offset** ainda é válido em listas pequenas e finitas: estados do Brasil, categorias de produto fixas, configurações.
- **Offset** quando a UI exige "ir para página exata N" (raríssimo fora de admin antigo). Considere mostrar busca por filtro em vez de jump.
- **Cursor bidirecional** quando o usuário precisa "voltar uma página" sem recarregar do início.
- **Não use cursor** se a ordenação muda dinamicamente por user input (ordem por preço asc/desc, alfabética). Cada mudança invalida cursores em uso — prefira oferecer "filtros" estáveis.

## Armadilhas comuns

- **Cursor sobre coluna não-única**: `created_at` sozinho. Se dois pedidos têm o mesmo timestamp (comum em inserts em batch), você pode pular ou repetir registros. Sempre componha com PK: `(created_at, id)`.
- **Index ausente sobre as colunas de ordenação**: query lenta apesar do cursor. Verifique `EXPLAIN` — deve mostrar `Index Scan` (Postgres) ou `Using index condition` (MySQL).
- **Expor cursor não opaco** (id puro): cliente fica acoplado ao schema. Refatorar a chave de paginação quebra apps.
- **Esquecer truque `LIMIT n+1`**: sem ele, você precisa de query extra `COUNT(*)` para saber se há mais — caro em tabelas grandes.
- **Limit sem clamp máximo**: cliente envia `?limit=100000` e derruba o banco. Sempre `min($limit, 100)`.
- **Cursor não invalida com mudança de ordering**: alternar `sort=created_at` para `sort=price` mantendo cursor antigo causa salto incoerente. Embeba o critério de ordenação dentro do cursor opaco.
- **Não tratar cursor expirado**: cursor de 6 meses atrás aponta para timestamp que ainda existe, mas dados foram migrados para tabela arquivo. Decida: aceitar tudo, ou expirar cursores com TTL embutido.
- **Confundir keyset com offset em SQL**: `WHERE id > 100 LIMIT 20 OFFSET 100` é o pior dos dois mundos. Use só o WHERE.

## Links relacionados

- [[REST-Maduro]]
- [[Performance/Fibers-N1]]
- [[Performance/Cache-PSR-6-16]]
- [[OpenAPI]]
- [[PSRs]]

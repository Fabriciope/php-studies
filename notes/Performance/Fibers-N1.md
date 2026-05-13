---
tags: [php, performance, fibers, async, n1]
fase: 6
status: draft
---

# Fibers + diagnóstico de N+1

## Conceito

**Fibers** (PHP 8.1+) são unidades de execução **cooperativa, single-thread**. Diferente de threads do SO, Fibers compartilham um único processo e CPU; a transição entre uma Fiber e outra acontece quando o código **explicitamente** chama `Fiber::suspend()`. Quem decide quando "ceder a vez" é a própria Fiber — daí o nome *cooperative concurrency*.

O modelo mental é simples: imagine uma função que pode pausar no meio, devolver o controle para quem a chamou, e ser retomada depois exatamente de onde parou — preservando variáveis locais, stack e contexto. Antes do PHP 8.1, isso só existia via `Generator` (que tinha limitações sérias: não pode propagar exceções de cima para baixo, não pode suspender através de chamadas aninhadas). Fibers eliminam essas limitações.

Fibers **não rodam em paralelo**. Elas resolvem **concorrência de I/O**: enquanto uma Fiber espera resposta de socket, outra pode executar. Para CPU-bound, Fibers não dão ganho — uma CPU continua sendo uma CPU. O ganho aparece quando você tem N requisições HTTP a fazer e cada uma esperaria 200ms: em serial seriam 2s; com Fibers cooperando sob um event loop, são ~250ms (paralelizando a espera).

**N+1** é o anti-padrão mais comum em ORMs e código que itera entidades carregando relações. Você lista N pedidos → para cada pedido, query separada para `customer` → total de 1 + N queries quando 2 bastariam (`SELECT FROM customers WHERE id IN (...)`). Em listas de 50 itens, 51 queries no lugar de 2. Em pico de tráfego, banco bate o teto de conexões.

Os dois temas convivem nesta nota porque ambos são sobre **eliminar tempo de espera ocioso**: Fibers cooperam para sobrepor I/O concorrente; eliminar N+1 elimina I/O redundante. O profissional moderno reconhece os dois e os trata com ferramentas certas — não tenta resolver N+1 com Fibers, nem usar eager loading onde Fibers seriam idiomáticas.

Bibliotecas que tiram proveito de Fibers em PHP: **Amphp v3** (HTTP cliente, MySQL, Redis async), **ReactPHP** (com adaptadores), **Revolt** (event loop base). Frameworks como Symfony e Laravel ainda assumem blocking; ferramentas como **Spiral / RoadRunner** integram melhor.

## Por que importa

**Cenário 1 — agregador de APIs**. Sua rota `/dashboard` precisa de dados de 5 microsserviços. Em código blocking: 5 × 200ms = 1s. Com Fibers + cliente HTTP async: ~210ms (a 5 simultâneas). Saiu de "lento" para "rápido" sem nenhum recurso adicional, só mudando o cliente.

**Cenário 2 — listagem de pedidos com clientes (N+1 clássico)**. Endpoint `/orders` lista 50 pedidos com nome do cliente. Sem cuidado: 51 queries (1 lista + 50 lookups). Com `IN (...)`: 2 queries. Em produção com 100 RPS: 5.100 queries/s sobe para 200 queries/s — 25× menos load no banco.

**Cenário 3 — webhook com fanout**. Você precisa notificar 20 sistemas externos. Sequencial em PHP-FPM com timeout de 30s: pior caso 30s × 20 = 600s, FPM já matou o request muito antes. Com Fibers + cliente HTTP cooperativo: ≈ tempo do mais lento (~30s) cabe no orçamento. (Mesmo melhor: empurre para fila — ver [[Filas]].)

Em todos os casos, a falta dessa ferramenta força workarounds: lançar processo separado, abrir thread em outra linguagem, esperar que o "back-end alternativo" (Swoole, Roadrunner) carregue o mundo. Fibers entregam concorrência onde antes era exclusividade desses runtimes.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// === Fiber pura: a API básica do PHP 8.1 ===
$fiber = new Fiber(function (string $msg): string {
    echo "início: $msg\n";
    $eco = Fiber::suspend('preciso de um sinal');  // pausa aqui
    echo "retomado com: $eco\n";
    return 'pronto';
});

$pedido = $fiber->start('olá');           // executa até suspend()
echo "fiber pediu: $pedido\n";            // "preciso de um sinal"
$resultado = $fiber->resume('vai');       // retoma e roda até return
echo "resultado: $resultado\n";           // "pronto"
```

```php
<?php
declare(strict_types=1);

// === Concorrência real: 3 GETs em paralelo via Amp ===
// composer require amphp/http-client
use Amp\Http\Client\HttpClientBuilder;
use Amp\Http\Client\Request;
use function Amp\async;
use function Amp\Future\await;

$client = HttpClientBuilder::buildDefault();

$urls = [
    'https://httpbin.org/delay/1',
    'https://httpbin.org/delay/1',
    'https://httpbin.org/delay/1',
];

$start = hrtime(true);

// async() cria uma Fiber internamente; await() agrupa
$futures = array_map(
    fn (string $url) => async(fn () => $client->request(new Request($url))->getBody()->buffer()),
    $urls
);
$bodies = await($futures);

$ms = (hrtime(true) - $start) / 1e6;
printf("3 GETs em %.0f ms (sequencial seriam ~3000ms)\n", $ms);
```

```php
<?php
declare(strict_types=1);

// === Detector simples de N+1 em desenvolvimento ===
final class QueryLogger
{
    /** @var list<array{sql: string, params: array<int|string, mixed>, at: float}> */
    private array $queries = [];

    public function record(string $sql, array $params): void
    {
        $this->queries[] = [
            'sql'    => $sql,
            'params' => $params,
            'at'     => microtime(true),
        ];
    }

    /** @return array<string, int> SQL normalizado → contagem */
    public function suspectN1(int $threshold = 5): array
    {
        $counts = [];
        foreach ($this->queries as $q) {
            // normaliza: troca valores por '?' para agrupar queries iguais
            $normalized = preg_replace("/'[^']*'|\d+/", '?', $q['sql']) ?? $q['sql'];
            $counts[$normalized] = ($counts[$normalized] ?? 0) + 1;
        }
        return array_filter($counts, fn (int $n): int => $n >= $threshold);
    }
}

// Em middleware/decorator de PDO: registre TODA query.
// No fim da request (dev only): se houver entradas, logue como WARNING
// para o dev investigar.
```

```php
<?php
declare(strict_types=1);

// === Anti-pattern: N+1 ===
$orders = $pdo->query('SELECT id, customer_id FROM orders LIMIT 50')->fetchAll();
foreach ($orders as $o) {
    // 50 queries adicionais!
    $stmt = $pdo->prepare('SELECT name FROM customers WHERE id = ?');
    $stmt->execute([$o['customer_id']]);
    $o['customer_name'] = $stmt->fetchColumn();
}

// === Correção 1: eager load com IN ===
$orders = $pdo->query('SELECT id, customer_id FROM orders LIMIT 50')->fetchAll();
$customerIds = array_column($orders, 'customer_id');
$placeholders = implode(',', array_fill(0, count($customerIds), '?'));
$customers = $pdo->prepare("SELECT id, name FROM customers WHERE id IN ($placeholders)");
$customers->execute($customerIds);
$byId = array_column($customers->fetchAll(), null, 'id');

foreach ($orders as &$o) {
    $o['customer_name'] = $byId[$o['customer_id']]['name'] ?? null;
}

// === Correção 2: JOIN no SQL (preferível quando read-only) ===
$rows = $pdo->query(
    'SELECT o.id, o.customer_id, c.name AS customer_name
     FROM orders o
     JOIN customers c ON c.id = o.customer_id
     LIMIT 50'
)->fetchAll();
```

```php
<?php
declare(strict_types=1);

// === DataLoader pattern: bufferiza pedidos e resolve em lote ===
// Útil em GraphQL ou casos onde "quem pede" não tem visão da lista total.
final class CustomerLoader
{
    /** @var array<int, true> */
    private array $pending = [];
    /** @var array<int, array{name: string}> */
    private array $loaded = [];

    public function __construct(private \PDO $pdo) {}

    public function queue(int $id): void { $this->pending[$id] = true; }

    public function load(int $id): array
    {
        if (isset($this->loaded[$id])) return $this->loaded[$id];

        if ($this->pending === []) {
            $this->queue($id);
        }

        $ids = array_keys($this->pending);
        $ph  = implode(',', array_fill(0, count($ids), '?'));
        $stmt = $this->pdo->prepare("SELECT id, name FROM customers WHERE id IN ($ph)");
        $stmt->execute($ids);

        foreach ($stmt->fetchAll(\PDO::FETCH_ASSOC) as $row) {
            $this->loaded[(int) $row['id']] = ['name' => $row['name']];
        }
        $this->pending = [];
        return $this->loaded[$id] ?? throw new \RuntimeException("customer $id not found");
    }
}
```

## Quando usar / quando não usar

- **Fibers** quando você tem **I/O independente paralelizável** (chamadas HTTP, leituras de banco em servidores diferentes, gravações em cache distribuído). Combine com Amp v3 ou bibliotecas que falam async.
- **Fibers em FPM clássico**: tecnicamente possível, mas o ganho real só aparece se sua app usa bibliotecas async (cliente HTTP, banco). Cliente sync dentro de Fiber **bloqueia** a Fiber inteira.
- **Fibers em runtime persistente** (RoadRunner / FrankenPHP / Octane): aí brilha, porque o event loop sobrevive entre requests.
- **Não use Fibers** para CPU-bound (cálculo, parsing grande, criptografia). Não há paralelismo — uma CPU continua sendo uma CPU.
- **Eager loading** (`IN (...)`, `JOIN`, ou ORM `with()`) sempre que você for iterar relacionamentos.
- **DataLoader** quando o consumidor é polimórfico (GraphQL resolver) e você não controla a iteração.
- **Read Model** dedicado (CQRS leve — ver [[../Arquitetura/Domain-Events-CQRS]]) quando a listagem precisa de muitas relações.

## Armadilhas comuns

- **`Fiber::suspend()` sem alguém para retomar = deadlock silencioso.** A Fiber dorme para sempre. Em prática, isso só se evita usando o event loop: nunca chame `suspend()` direto fora de framework async. Use `delay()` ou `await()` da Amp, que se integram ao loop.
- **Bibliotecas síncronas dentro de Fiber bloqueiam tudo.** PDO sync, `file_get_contents()` HTTP, Guzzle sync — todos prendem a Fiber inteira até retornar. O event loop não pode "preemptar" porque é cooperativo. Use cliente HTTP da Amp, conexão Amp\Mysql, etc.
- **Fibers ≠ threads. Não compartilham memória entre processos.** Variável global em Fiber A é a mesma vista por Fiber B porque rodam no mesmo processo PHP. Não há corrida de dados (não há paralelismo de CPU), mas há corrida de **estado** entre `suspend` e `resume` — `$x = $api->get(); ...; $api->post($x)`, se algo entre `suspend/resume` mudar `$x`, surpresa.
- **N+1 escondido por lazy loading do ORM.** Doctrine/Eloquent fazem `getCustomer()` virar query quando acessado. O código parece limpo (`$order->customer->name`) mas dispara N queries. Ative SQL logger em dev e revise.
- **`fetchAll` em listas gigantes esgota memória.** Para listas grandes, use `fetch` em loop ou cursor com `unbuffered queries`. N+1 corrigido com `IN (10000 ids)` também pode estourar `max_allowed_packet` do MySQL.
- **Eager loading excessivo é o oposto do N+1, mas igualmente ruim.** Carregar 15 relações sempre "por garantia" traz dados que não serão usados — vira *over-fetching*. Carregue só o que a view consome.
- **`array_column` com chaves não únicas perde dados.** `array_column($rows, null, 'customer_id')` quando há dois pedidos do mesmo customer mantém só um. Para casos M:N, agrupe manualmente.
- **N+1 com cache "esconde" o problema.** Primeiro request enche cache, próximos são rápidos. Mas em deploy/restart, a próxima onda derruba banco. Resolva N+1 mesmo com cache.
- **Confundir `Fiber` com `Generator`.** Generators (yield) também pausam e retomam, mas só uma direção: caller controla. Fibers permitem qualquer um suspender — caller ou callee. Em código novo, Fibers > Generators para concorrência.

## Cheatsheet de comandos

```bash
# Detectar N+1 com query log do MySQL em dev
SET GLOBAL general_log = 'ON';
SET GLOBAL general_log_file = '/tmp/mysql.log';
# rode o cenário, depois:
sort /tmp/mysql.log | uniq -c | sort -rn | head

# Em Doctrine, force erro se houver query em loop
$config->setSQLLogger(new \Doctrine\DBAL\Logging\DebugStack());
# inspecione $logger->queries no fim da request

# Em Laravel, use \DB::listen() e \DB::enableQueryLog()
```

## Links relacionados

- [[FrankenPHP-RoadRunner-Swoole]]
- [[Profiling]]
- [[Cache-PSR-6-16]]
- [[Filas]]
- [[../Padroes/Repository]]
- [[../Arquitetura/Domain-Events-CQRS]]

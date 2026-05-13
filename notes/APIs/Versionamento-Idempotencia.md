---
tags: [php, api, versionamento, idempotencia]
fase: 5
status: draft
---

# Versionamento & Idempotência

## Conceito

São dois temas independentes que vivem juntos porque ambos protegem contratos no tempo: versionamento protege contra **mudanças no servidor**; idempotência protege contra **mudanças na rede**.

### Versionamento

Toda API pública (e a maioria das internas) vai sofrer breaking changes — campo removido, tipo alterado, semântica diferente. Versionamento é a estratégia que mantém clientes antigos vivos enquanto a API evolui. Três estilos dominam:

- **URL path** — `https://api.example.com/v1/orders`. Mais comum, visível em logs, fácil de routear em proxy/CDN, fácil de explicar a parceiros. Trade-off: rompe ideia REST pura de que "um recurso tem um URI", já que `/v1/orders/123` e `/v2/orders/123` apontam para a "mesma entidade".
- **Header customizado** — `Accept-Version: 2` ou `X-API-Version: 2`. Mantém URI limpa, mas é invisível em logs comuns e debugging com curl exige flag extra.
- **Media type customizado (vendor type)** — `Accept: application/vnd.example.v2+json`. Tecnicamente o mais "RESTful" segundo Fielding (negociação por representação), suportado pela RFC 6838. Trade-off: clientes esquecem de mandar e caem na default; ferramentas como Swagger UI suportam mal.
- **Query string** — `?version=2`. Desencorajado: pollui cache key, dificulta routing.

| Estilo | Visibilidade | Compatibilidade CDN | Adoção |
| --- | --- | --- | --- |
| URL path | Alta | Fácil | GitHub, Stripe, Twilio |
| Header | Baixa | Requer Vary | Heroku, GitHub (parcial) |
| Vendor media type | Média | Requer Vary | GitHub (atual), Atlassian |
| Query string | Alta | Polui chave | Desencorajado |

**Deprecação** é parte do versionamento. RFC 8594 padronizou o header `Sunset: Sat, 31 Dec 2026 23:59:59 GMT` para anunciar quando uma versão será descontinuada. Combinado com `Deprecation: true` (draft em discussão) e `Link: <https://api.example.com/v2>; rel="successor-version"`, dá aviso programático ao cliente.

Janela mínima de deprecação varia por contexto: 6 meses para APIs públicas grandes, 12 meses para integrações B2B, ad-hoc para APIs internas. Comunicação ativa (e-mail, dashboard) é tão importante quanto o header.

### Idempotência

Uma operação é **idempotente** se executá-la N vezes produz o mesmo estado final que executá-la 1 vez. `GET`, `PUT` e `DELETE` são naturalmente idempotentes pela semântica HTTP. `POST` não é.

O problema clássico: rede instável. Cliente envia `POST /charges` (cobrança), servidor processa e cria cobrança, mas resposta se perde. Cliente faz retry — sem proteção, gera segunda cobrança. Em sistemas financeiros isso é catastrófico.

A solução padrão (popularizada pela Stripe em 2015) é o header **`Idempotency-Key`**: cliente gera um UUID, envia em toda tentativa do mesmo POST. Servidor:

1. Lê a chave.
2. Verifica em cache (Redis tipicamente, TTL ~24h).
3. Se cache hit, retorna a **resposta original** (mesmo status, mesmo body).
4. Se cache miss, processa, salva resposta no cache, retorna.

Detalhes críticos:
- Cache hash deve incluir **método + path + key**, não apenas key. Caso contrário cliente reusando key acidentalmente em endpoint diferente recebe resposta errada.
- **Concorrência**: dois retries chegam ao mesmo tempo (rede com jitter). Sem lock, os dois processam. Solução: Redis `SET key value NX EX 86400` cria entry exclusiva; segundo cliente vê "in progress" e espera ou recebe 409.
- **Resposta a cachear**: só 2xx e 4xx **determinísticos** (422 de validação, 409 de conflito). Nunca 5xx — erro transitório do servidor deve permitir retry.
- **Comparação de payload**: bom padrão extra é comparar hash do body. Se cliente reusa key com body diferente, retorna 422 "key already used with different payload".

RFC 9110 §9.2.2 reforça a semântica. Existe um IETF draft (`draft-ietf-httpapi-idempotency-key-header`) padronizando o header.

## Por que importa

Sem versionamento, qualquer breaking change quebra apps em produção, no celular do cliente, fora do seu controle. Não dá para "forçar atualização" de um app mobile que está em 50% dos usuários ainda na versão antiga. Versionamento explícito permite manter v1 viva por meses enquanto v2 ganha tração.

Sem idempotência, retries (humanos clicando duas vezes, mobile em 3G oscilante, gateway de pagamento com timeout interno) duplicam pedidos, cobranças, e-mails, mensagens. O desastre é **invisível** — não há log de erro, apenas dados duplicados que aparecem em reclamação de cliente. Times só descobrem com auditoria.

Cenário concreto 1 — **deploy quebra cliente mobile**: equipe remove campo `legacy_id` do payload de pedido. App mobile v1.4 (40% da base) lê o campo e crasha. Com versionamento, v1 continua expondo o campo; v2 não tem. App migra quando puder.

Cenário concreto 2 — **double charge no Black Friday**: pico de tráfego, gateway responde lento, cliente faz retry, segunda cobrança vai. Time descobre 2 dias depois via NPS negativo. Com `Idempotency-Key`, retry é seguro.

Cenário concreto 3 — **webhook de parceiro com retries**: parceiro reenvia mesmo webhook 5 vezes em 1 hora porque seu timeout é agressivo. Sem dedup, você processa 5x o mesmo evento. Use a key (ou `event_id` do parceiro) como `Idempotency-Key`.

## Exemplo de código PHP

Exemplo 1 — middleware PSR-15 de idempotência com Redis e lock:

```php
<?php
declare(strict_types=1);

use Psr\Http\Message\{ResponseInterface, ServerRequestInterface};
use Psr\Http\Server\{MiddlewareInterface, RequestHandlerInterface};
use Psr\SimpleCache\CacheInterface;

final readonly class IdempotencyMiddleware implements MiddlewareInterface
{
    public function __construct(
        private CacheInterface $cache,
        private ResponseFactoryInterface $responses,
        private int $ttl = 86400, // 24h
    ) {}

    public function process(
        ServerRequestInterface $req,
        RequestHandlerInterface $next,
    ): ResponseInterface {
        // só métodos não idempotentes precisam de proteção
        if (!in_array($req->getMethod(), ['POST', 'PATCH'], true)) {
            return $next->handle($req);
        }

        $key = $req->getHeaderLine('Idempotency-Key');
        if ($key === '') {
            return $next->handle($req);
        }
        if (!preg_match('/^[A-Za-z0-9_\-]{8,128}$/', $key)) {
            return $this->problem(400, 'invalid Idempotency-Key format');
        }

        // hash inclui método + path para isolar escopos
        $hash = sprintf(
            'idem:%s:%s:%s',
            $req->getMethod(),
            $req->getUri()->getPath(),
            hash('sha256', $key),
        );

        $cached = $this->cache->get($hash);
        if ($cached !== null) {
            return $this->rebuild($cached)
                ->withHeader('Idempotent-Replayed', 'true');
        }

        // SETNX pseudo: só processa se conseguir gravar marca de in-progress
        $lockKey = $hash . ':lock';
        if (!$this->tryLock($lockKey, ttl: 30)) {
            return $this->problem(409, 'request in progress');
        }

        try {
            $res = $next->handle($req);
            if ($res->getStatusCode() < 500) {
                $this->cache->set($hash, $this->serialize($res), $this->ttl);
            }
            return $res->withHeader('Idempotent-Replayed', 'false');
        } finally {
            $this->cache->delete($lockKey);
        }
    }

    private function tryLock(string $key, int $ttl): bool { /* Redis SET NX EX */ return true; }
    private function serialize(ResponseInterface $r): array { /* ... */ return []; }
    private function rebuild(array $data): ResponseInterface { /* ... */ return $this->responses->createResponse(); }
    private function problem(int $status, string $title): ResponseInterface { /* Problem Details */ return $this->responses->createResponse($status); }
}
```

Exemplo 2 — versionamento por URL path com router PSR-15:

```php
<?php
declare(strict_types=1);

final readonly class VersionedRouter
{
    /** @param array<string, array<string, callable>> $routes */
    public function __construct(private array $routes) {}

    public function dispatch(ServerRequestInterface $req): ResponseInterface
    {
        $path = $req->getUri()->getPath();
        if (!preg_match('#^/v(\d+)(/.*)$#', $path, $m)) {
            throw new NotFoundException();
        }
        [$_, $version, $rest] = $m;

        $handlers = $this->routes['v' . $version] ?? null;
        if ($handlers === null) {
            throw new GoneException(410, "API v{$version} no longer supported");
        }

        $handler = $handlers[$req->getMethod() . ' ' . $rest] ?? throw new NotFoundException();
        return $handler($req);
    }
}
```

Exemplo 3 — anúncio de deprecação via headers:

```php
<?php
declare(strict_types=1);

final readonly class DeprecationMiddleware implements MiddlewareInterface
{
    public function __construct(
        private \DateTimeImmutable $sunset,
        private string $successorUrl,
    ) {}

    public function process(
        ServerRequestInterface $req,
        RequestHandlerInterface $next,
    ): ResponseInterface {
        $res = $next->handle($req);
        return $res
            ->withHeader('Deprecation', 'true')
            ->withHeader('Sunset', $this->sunset->format(\DateTimeInterface::RFC7231))
            ->withHeader('Link', "<{$this->successorUrl}>; rel=\"successor-version\"");
    }
}
```

Exemplo 4 — combinação idempotência + transação de banco:

```php
<?php
declare(strict_types=1);

final readonly class CreateOrderAction
{
    public function __construct(
        private \PDO $pdo,
        private CacheInterface $cache,
    ) {}

    public function execute(string $idemKey, array $payload): Order
    {
        // dedup ao nível do banco também — defesa em profundidade
        $hash = hash('sha256', $idemKey . json_encode($payload));

        $this->pdo->beginTransaction();
        try {
            $stmt = $this->pdo->prepare(
                'INSERT INTO idempotency_log (hash, created_at) VALUES (?, NOW())
                 ON CONFLICT (hash) DO NOTHING RETURNING id'
            );
            $stmt->execute([$hash]);
            if ($stmt->fetch() === false) {
                $this->pdo->rollBack();
                throw new ConflictException('order already submitted');
            }

            $order = Order::create($payload);
            // persistir...
            $this->pdo->commit();
            return $order;
        } catch (\Throwable $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }
}
```

## Quando usar / quando não usar

- **Versionamento URL path** — default para APIs públicas e B2B. Visível, fácil de routear, fácil de explicar.
- **Versionamento por header/media-type** — quando o domínio é polimórfico (diferentes recursos evoluem em ritmos diferentes) ou quando o time controla cliente e servidor.
- **Sem versionamento explícito** — APIs internas com deploy coordenado de cliente+servidor (mono-repo, deploy atômico).
- **`Idempotency-Key` obrigatório** — em POSTs que criam recursos com efeito externo: cobrança, envio físico, e-mail, notificação push, criação de invoice.
- **`Idempotency-Key` opcional** — em POSTs de leitura cara mas sem efeito colateral (relatórios).
- **Não use idempotência** em GETs (já são idempotentes por definição).
- **Não use idempotência em PATCH** se o payload é diff parcial — combinar duas chamadas idênticas pode não fazer sentido (depende da semântica).

## Armadilhas comuns

- **Cachear resposta 5xx**: cliente recebe o erro para sempre durante o TTL. Cacheie apenas 2xx e 4xx determinísticos.
- **Hash do cache de idempotência apenas com a key**: dois endpoints diferentes usando a mesma key produzem colisão. Sempre inclua método + path no hash.
- **TTL muito curto** (1h): cliente que faz retry tardio (job batch após 12h) cria duplicata. Padrão Stripe é 24h.
- **TTL muito longo** (30 dias): cache cresce demais, e cliente que reusa key por engano em integração nova recebe resposta antiga.
- **Sem lock contra concorrência**: dois retries simultâneos passam pelo `cache->get` ao mesmo tempo, ambos veem miss, ambos processam. Use SETNX/SET NX EX.
- **Não validar formato da key**: cliente envia 5MB de "key", você grava em Redis. Aceite apenas regex razoável (UUID, 8-128 chars alfanuméricos).
- **Deprecação sem aviso**: removeu v1 em deploy de quinta-feira, parceiros descobriram na sexta. Sempre `Sunset` header + comunicação ativa com janela mínima de 6 meses.
- **Quebrar contrato dentro da mesma versão major**: adicionar campo opcional é OK, remover não é. Em v1.x mantenha retrocompatibilidade estrita.
- **Confundir idempotência com unicidade de negócio**: `Idempotency-Key` protege contra retry de rede, não contra "usuário clicou em pedir 2 vezes intencionalmente". Para isso use chave de domínio (carrinho_id) em deduplicação de aplicação.

## Links relacionados

- [[PSRs]]
- [[REST-Maduro]]
- [[OpenAPI]]
- [[JWT-vs-Sessoes]]
- [[Performance/Cache-PSR-6-16]]
- [[Infra/Observabilidade-OWASP]]

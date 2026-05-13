---
tags: [php, api, rest]
fase: 5
status: draft
---

# REST maduro

## Conceito

REST (Representational State Transfer) é o estilo arquitetural definido por Roy Fielding em sua dissertação de doutorado de 2000 ("Architectural Styles and the Design of Network-based Software Architectures", capítulo 5). Não é um protocolo nem um padrão fechado: é um conjunto de **constraints** (cliente-servidor, stateless, cacheável, interface uniforme, sistema em camadas, code-on-demand opcional) que, quando aplicados sobre HTTP, produzem APIs que aproveitam ao máximo a infraestrutura web existente (caches, proxies, balanceadores).

Em 2008, Leonard Richardson popularizou o **Richardson Maturity Model**, uma escala didática de 0 a 3 que ajuda a avaliar o quão "RESTful" uma API realmente é:

- **Nível 0 — The Swamp of POX**: HTTP como tunel RPC. Um único endpoint (`POST /api`), corpo dita a ação. SOAP clássico vive aqui.
- **Nível 1 — Recursos**: cada entidade tem URI própria (`/orders/123`, `/users/45`), mas tudo ainda é POST e status code é 200 mesmo em erro.
- **Nível 2 — Verbos HTTP + status codes**: GET é seguro, PUT/DELETE são idempotentes, POST cria, status code reflete semântica (201, 404, 409, 422). É onde a esmagadora maioria das APIs profissionais para.
- **Nível 3 — HATEOAS** (Hypermedia As The Engine Of Application State): o servidor envia links no payload (`{"_links": {"cancel": "/orders/123/cancel"}}`). Cliente navega por links em vez de hardcodear URIs. É o "REST verdadeiro" segundo Fielding, raramente implementado de forma completa.

Acima dos verbos vivem os métodos HTTP categorizados pela RFC 9110 (que unificou em 2022 as antigas RFC 7230-7235):

- **Safe** (não muda estado observável): `GET`, `HEAD`, `OPTIONS`, `TRACE`.
- **Idempotent** (N execuções = 1 execução em efeito): `GET`, `HEAD`, `OPTIONS`, `PUT`, `DELETE`.
- **Cacheable** por default: `GET`, `HEAD`; condicionalmente `POST` (raramente usado).

Note que `POST` é o único método "comum" que não é idempotente nem safe — é por isso que ele exige `Idempotency-Key` quando criar recursos com efeito externo (ver [[Versionamento-Idempotencia]]).

Negociação de conteúdo (content negotiation, RFC 9110 §12) permite ao cliente pedir representação específica via `Accept: application/json`, `Accept-Language: pt-BR`, `Accept-Encoding: gzip`. O servidor responde com `Content-Type` correspondente e `Vary` listando os headers que afetaram a resposta — essencial para caches.

Caching condicional usa `ETag` (hash do recurso) e `Last-Modified`. Cliente reenvia com `If-None-Match: "abc123"` ou `If-Modified-Since: ...`; servidor responde `304 Not Modified` sem corpo se nada mudou. Para escrita otimista, `If-Match` previne lost-update: cliente diz "atualize apenas se versão ainda for X"; servidor retorna `412 Precondition Failed` se houve alteração concorrente.

Erros padronizados seguem **RFC 7807 / 9457 — Problem Details for HTTP APIs**: corpo `application/problem+json` com campos `type` (URI), `title`, `status`, `detail`, `instance` e extensões livres. É o equivalente a um exception type comum entre serviços.

## Por que importa

REST não é só "JSON sobre HTTP". Verbo errado, status errado e payload sem padrão fazem cada cliente reimplementar lógica em cada endpoint. REST maduro entrega **previsibilidade**: o cliente sabe que `404` é "não existe", `409` é "conflito", `PUT` é idempotente, e ele pode usar a infraestrutura HTTP (proxies, CDNs, load balancers) sem surpresa.

Cenário concreto 1 — **CDN não consegue cachear**: a API retorna 200 com `{"error": "not found"}` em vez de 404. CloudFront/Cloudflare/Varnish cacheiam o "erro" como sucesso e servem para todos. A migração para status code correto reduz origem em 60-80% no padrão de tráfego típico.

Cenário concreto 2 — **mobile retry duplica pedido**: rede oscila, cliente reenvia POST. Sem idempotência e sem 201+Location, o backend cria dois pedidos. Com PUT em recurso já nomeado pelo cliente (`PUT /orders/{client-uuid}`), retry é seguro por definição.

Cenário concreto 3 — **front-end faz polling**: 1000 clientes consultam `/me/notifications` a cada 5s. Sem ETag, são 1000 respostas full a cada 5s. Com ETag + `If-None-Match`, 95% retornam 304 sem corpo — banda cai 90%, p99 melhora.

Cenário concreto 4 — **erro em integração B2B**: parceiro recebe 500 com HTML de stack trace. Equipe deles não sabe se é problema deles ou seu, abre ticket. Com Problem Details (`type: "https://api.example/errors/invoice-locked"`), parceiro programa o handling automaticamente.

## Exemplo de código PHP

Exemplo 1 — estrutura de rotas e mapeamento HTTP correto:

```php
<?php
declare(strict_types=1);

// recursos como substantivos plurais; verbos via método HTTP
//
// GET    /v1/orders            list          200
// POST   /v1/orders            create        201 + Location
// GET    /v1/orders/{id}       show          200 / 404
// PUT    /v1/orders/{id}       replace       200 / 201 / 404
// PATCH  /v1/orders/{id}       partial       200 / 404 / 422
// DELETE /v1/orders/{id}       cancel        204 / 404
// GET    /v1/orders/{id}/items nested        200 / 404
```

Exemplo 2 — Problem Details RFC 9457:

```php
<?php
declare(strict_types=1);

use Psr\Http\Message\ResponseInterface;

final readonly class Problem
{
    public function __construct(
        public string $type,           // "https://api.example/errors/payment-declined"
        public string $title,          // "Payment declined"
        public int $status,            // 422
        public ?string $detail = null,
        public ?string $instance = null,
        public array $extensions = [], // ex: ['retry_after' => 30]
    ) {}

    public function toResponse(ResponseFactoryInterface $f): ResponseInterface
    {
        $body = array_filter([
            'type'     => $this->type,
            'title'    => $this->title,
            'status'   => $this->status,
            'detail'   => $this->detail,
            'instance' => $this->instance,
            ...$this->extensions,
        ], fn ($v) => $v !== null);

        return $f->createResponse($this->status)
            ->withHeader('Content-Type', 'application/problem+json')
            ->withBody(/* stream com json_encode($body) */);
    }
}
```

Exemplo 3 — ETag + concorrência otimista (If-Match):

```php
<?php
declare(strict_types=1);

final readonly class OrderController
{
    public function __construct(private OrderRepository $repo) {}

    public function update(ServerRequestInterface $req, string $id): ResponseInterface
    {
        $order = $this->repo->find($id) ?? throw new NotFoundException();
        $etag  = '"' . sha1($order->version . $order->id) . '"';

        $ifMatch = $req->getHeaderLine('If-Match');
        if ($ifMatch !== '' && $ifMatch !== $etag) {
            // alguém atualizou no meio do caminho
            throw new PreconditionFailedException(412, 'order changed');
        }

        $data = json_decode((string) $req->getBody(), true);
        $updated = $this->repo->update($order, $data);
        $newEtag = '"' . sha1($updated->version . $updated->id) . '"';

        return (new Response(200))
            ->withHeader('ETag', $newEtag)
            ->withHeader('Cache-Control', 'private, must-revalidate');
    }
}
```

Exemplo 4 — content negotiation manual:

```php
<?php
declare(strict_types=1);

function negotiate(ServerRequestInterface $req): string
{
    $accept = $req->getHeaderLine('Accept');
    return match (true) {
        str_contains($accept, 'application/xml')      => 'xml',
        str_contains($accept, 'text/csv')             => 'csv',
        str_contains($accept, 'application/json'),
        $accept === '' || $accept === '*/*'           => 'json',
        default => throw new NotAcceptableException(406),
    };
}
```

## Status codes — guia completo

| Faixa | Significado | Casos comuns |
| --- | --- | --- |
| 1xx | Informacional | 100 Continue (uploads grandes), 101 Switching Protocols (WebSocket) |
| 200 | OK | GET/PUT/PATCH com corpo de resposta |
| 201 | Created | POST que cria recurso. **Sempre** com `Location:` |
| 202 | Accepted | Processamento assíncrono iniciado |
| 204 | No Content | DELETE bem sucedido, PUT sem body de retorno |
| 206 | Partial Content | Range requests (download parcial, vídeo) |
| 301/308 | Permanent redirect | URI mudou para sempre (308 preserva método) |
| 302/307 | Temporary redirect | Manutenção, A/B (307 preserva método) |
| 304 | Not Modified | Cache hit via ETag/If-Modified-Since |
| 400 | Bad Request | JSON malformado, parâmetros faltando |
| 401 | Unauthorized | Sem credenciais ou inválidas |
| 403 | Forbidden | Autenticado, mas sem permissão |
| 404 | Not Found | Recurso inexistente OU oculto por autorização |
| 405 | Method Not Allowed | Verbo errado no endpoint. Inclua `Allow:` header |
| 406 | Not Acceptable | `Accept` não satisfeito |
| 409 | Conflict | Conflito de estado (versão obsoleta) |
| 410 | Gone | Recurso existiu e foi removido permanentemente |
| 412 | Precondition Failed | `If-Match` não bateu |
| 415 | Unsupported Media Type | `Content-Type` inválido no body |
| 422 | Unprocessable | Sintaxe OK, mas semântica inválida (validação) |
| 425 | Too Early | Replay protection (TLS 0-RTT) |
| 428 | Precondition Required | Servidor exige `If-Match` |
| 429 | Too Many Requests | Rate limit. Inclua `Retry-After:` |
| 500 | Internal Server Error | Bug do servidor. **Nunca** vaze stack trace |
| 502 | Bad Gateway | Upstream retornou inválido |
| 503 | Service Unavailable | Sobrecarga ou manutenção. Inclua `Retry-After:` |
| 504 | Gateway Timeout | Upstream demorou demais |

## Quando usar / quando não usar

- **REST nível 2** — default para APIs públicas e internas. Cobertura ampla, ferramental maduro (OpenAPI, Postman, curl).
- **REST nível 3 (HATEOAS)** — quando há fluxo de trabalho complexo (state machine de pedido, workflows BPMN) e clientes que se beneficiam de descobrir transições válidas dinamicamente.
- **GraphQL** — quando há overfetch crônico (mobile pagando 3G), gráfico de dados muito heterogêneo, ou múltiplos consumidores cada um pedindo um subset diferente.
- **gRPC** — comunicação interna entre microsserviços, alta frequência, tipos fortes; sobre HTTP/2 com Protobuf.
- **JSON-RPC** — quando o domínio é naturalmente procedural (RPC) e REST fica forçado.
- **Não invente** "POST /createOrder" ou "POST /api?action=create". Verbos vão no método HTTP.
- **Não exponha** modelos de banco direto como recursos REST. Crie representações intencionais.

## Armadilhas comuns

- **Retornar 200 com `{"error": "..."}`** — cliente que olha só status code acha que deu certo. CDN cacheia o erro. Sempre status correto + corpo Problem Details.
- **DELETE não idempotente** — DELETE 2x na mesma URI deve produzir o mesmo estado final (recurso ausente). Retornar 404 na segunda é aceitável; retornar 500 não.
- **POST tratado como idempotente sem `Idempotency-Key`** — retry de cliente cria duplicatas silenciosas. Use header explícito ou prefira PUT com URI determinada pelo cliente.
- **404 vs 403 vazando informação** — em recursos protegidos, retornar 404 quando o usuário não tem permissão evita revelar existência. Use 404 consistente em vez de 403.
- **PATCH com semântica inventada** — RFC 5789 define PATCH como aplicação de diff. Use `application/merge-patch+json` (RFC 7396) ou `application/json-patch+json` (RFC 6902); não JSON arbitrário com convenção privada.
- **Status 200 em criação** — quebra clientes que esperam 201+Location para saber a URI nova do recurso.
- **Vary header esquecido** — resposta varia com Accept-Language mas Vary não lista, CDN serve português para usuário em inglês.
- **Cache-Control: no-cache mal interpretado** — `no-cache` significa "revalide com ETag", não "não cacheie". Para impedir cache use `no-store`.

## Links relacionados

- [[PSRs]]
- [[Versionamento-Idempotencia]]
- [[OpenAPI]]
- [[Paginacao-Cursor]]
- [[JWT-vs-Sessoes]]
- [[Performance/Cache-PSR-6-16]]

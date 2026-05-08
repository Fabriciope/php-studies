---
tags: [php, api, rest]
fase: 5
status: stub
---

# REST maduro

## Conceito

Richardson Maturity Model define 4 níveis de "REST":

- **0** — HTTP como tunel RPC (`POST /api`)
- **1** — recursos com URI próprios (`/orders/123`)
- **2** — uso correto de verbos e status (GET seguro/idempotente, POST cria, PUT idempotente, DELETE idempotente, 4xx vs 5xx)
- **3** — HATEOAS (links no payload)

A maioria dos sistemas profissionais para no nível 2 e ali já é "REST maduro o suficiente".

## Por que importa

REST não é só "JSON sobre HTTP". Verbo errado, status errado e payload sem padrão fazem o cliente reimplementar lógica em cada endpoint. REST maduro entrega **previsibilidade**: o cliente sabe que `404` é "não existe", `409` é "conflito", `PUT` é idempotente.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// recursos como substantivos plurais; verbos via método HTTP
//
// GET    /v1/orders            list
// POST   /v1/orders            create        201 + Location
// GET    /v1/orders/{id}       show          200 / 404
// PATCH  /v1/orders/{id}       partial       200 / 404 / 422
// DELETE /v1/orders/{id}       cancel        204 / 404
//
// erros padronizados (RFC 9457 Problem Details)

final readonly class Problem {
    public function __construct(
        public string $type,      // "https://api.example/errors/payment-declined"
        public string $title,     // "Payment declined"
        public int $status,       // 422
        public ?string $detail = null,
        public ?string $instance = null,
        public array $extensions = [],
    ) {}

    public function toResponse(): ResponseInterface {
        $body = [
            'type' => $this->type,
            'title' => $this->title,
            'status' => $this->status,
            'detail' => $this->detail,
            'instance' => $this->instance,
            ...$this->extensions,
        ];
        return new Response($this->status, [
            'Content-Type' => 'application/problem+json',
        ], json_encode(array_filter($body, fn ($v) => $v !== null)));
    }
}
```

## Status codes — guia rápido

- **200** — sucesso com body
- **201** — criou recurso (use `Location: /v1/orders/123`)
- **202** — aceito, processando assíncrono
- **204** — sucesso sem body (DELETE)
- **400** — requisição malformada
- **401** — não autenticado
- **403** — autenticado mas não autorizado
- **404** — recurso inexistente
- **409** — conflito (versionamento otimista)
- **422** — entendido mas semanticamente inválido (validação)
- **429** — rate limit
- **500** — erro do servidor (NUNCA exponha stack)
- **503** — sobrecarga / manutenção

## Quando usar / quando não usar

- **REST nível 2** — default para APIs públicas/internas.
- **GraphQL/RPC** — quando há overfetch crônico ou domínio muito heterogêneo.
- **Não invente** "POST /createOrder". Verbos vão no método HTTP.

## Armadilhas comuns

- Retornar 200 com `{"error": "..."}` — cliente acha que deu certo. Use status correto.
- DELETE não idempotente — fazer DELETE 2x retornar 404 na 2ª é OK, mas o efeito **deve** ser o mesmo.
- POST idempotente sem `Idempotency-Key` — você precisa de proteção explícita (ver [[Versionamento-Idempotencia]]).

## Links relacionados

- [[PSRs]]
- [[Versionamento-Idempotencia]]
- [[OpenAPI]]

---
tags: [php, api, versionamento, idempotencia]
fase: 5
status: stub
---

# Versionamento & Idempotência

## Conceito

**Versionamento** mantém clientes antigos vivos enquanto a API evolui. Três estilos:

- **URL path** — `/v1/orders` (mais comum, mais visível)
- **Header** — `Accept: application/vnd.api+json; version=1`
- **Content type custom** — `Accept: application/vnd.example.v2+json`

**Idempotência** garante que enviar o mesmo POST 2x produz o **mesmo efeito** que enviar 1x. Implementa-se com `Idempotency-Key` (UUID gerado pelo cliente): servidor armazena resultado da primeira execução e replica nas próximas.

## Por que importa

Sem versionamento, qualquer breaking change quebra apps em produção. Sem idempotência, retry de cliente em rede instável duplica pedidos, cobranças, e-mails — desastre invisível.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// idempotência via PSR-15 + cache
use Psr\SimpleCache\CacheInterface;

final readonly class IdempotencyMiddleware implements MiddlewareInterface {
    public function __construct(
        private CacheInterface $cache,
        private int $ttl = 86400, // 24h
    ) {}

    public function process(
        ServerRequestInterface $req,
        RequestHandlerInterface $next,
    ): ResponseInterface {
        if (!in_array($req->getMethod(), ['POST', 'PATCH'], true)) {
            return $next->handle($req);
        }
        $key = $req->getHeaderLine('Idempotency-Key');
        if ($key === '') {
            return $next->handle($req);
        }

        $hash = "idem:{$req->getMethod()}:{$req->getUri()->getPath()}:{$key}";
        if ($cached = $this->cache->get($hash)) {
            return $this->rebuild($cached);
        }

        // lock simples para evitar concorrência (em prod: Redis SETNX)
        $res = $next->handle($req);

        // só cacheia respostas de sucesso ou 4xx determinísticos (não 5xx)
        if ($res->getStatusCode() < 500) {
            $this->cache->set($hash, $this->serialize($res), $this->ttl);
        }
        return $res->withHeader('Idempotent-Replayed', 'false');
    }
    // serialize/rebuild omitidos
}
```

## Quando usar / quando não usar

- **Versionamento URL path** — default. Visível, fácil de routear.
- **Versionamento header** — quando o domínio é polimórfico e versão varia por recurso.
- **Idempotency-Key obrigatório** em POST que cria recursos com efeito externo (cobrança, envio, e-mail).
- **Não usar idempotência** em GETs (já são idempotentes por definição).

## Armadilhas comuns

- Cachear resposta 5xx — cliente vai receber o erro pra sempre. Cacheie só ≤4xx.
- Idempotency-Key no escopo errado: usar a **mesma key** para POST diferente (path + método diferentes) leva a comportamento estranho. Inclua método + path no hash.
- Versionamento sem deprecation policy — deprecar v1 sem avisar quebra parceiros. Defina janela mínima (6–12 meses).
- TTL do cache de idempotência muito curto — cliente que faz retry após 25h cria duplicata.

## Links relacionados

- [[PSRs]]
- [[REST-Maduro]]
- [[Performance/Cache-PSR-6-16]]

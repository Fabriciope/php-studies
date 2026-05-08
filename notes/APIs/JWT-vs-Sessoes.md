---
tags: [php, api, auth, jwt, sessao]
fase: 5
status: stub
---

# JWT vs Sessões

## Conceito

**Sessão**: servidor guarda o estado (em Redis/banco), cliente recebe um cookie opaco (session id). Toda requisição: lookup pelo id.

**JWT**: token autocontido assinado (header.payload.signature). Servidor valida assinatura e usa claims sem lookup. Stateless.

## Por que importa

Escolher errado custa. JWT em monólito web tradicional é overkill com problema de logout. Sessões em arquitetura distribuída multi-cliente (mobile + SPA + parceiros) viram pesadelo de sticky-session. Decisão é **arquitetural**, não preferência.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// JWT — assinatura HS256 manual (em prod use firebase/php-jwt ou paragonie/paseto)
final readonly class Jwt {
    public function __construct(private string $secret) {}

    public function issue(array $claims, int $ttl = 900): string {
        $header  = $this->b64(json_encode(['alg' => 'HS256', 'typ' => 'JWT']));
        $payload = $this->b64(json_encode([
            ...$claims,
            'iat' => time(),
            'exp' => time() + $ttl,
            'jti' => bin2hex(random_bytes(8)),
        ]));
        $sig = $this->b64(hash_hmac('sha256', "{$header}.{$payload}", $this->secret, true));
        return "{$header}.{$payload}.{$sig}";
    }

    public function verify(string $token): array {
        [$h, $p, $s] = explode('.', $token, 3) + [null, null, null];
        if (!$h || !$p || !$s) throw new \RuntimeException('malformed');
        $expected = $this->b64(hash_hmac('sha256', "{$h}.{$p}", $this->secret, true));
        if (!hash_equals($expected, $s)) throw new \RuntimeException('invalid sig');
        $claims = json_decode(base64_decode(strtr($p, '-_', '+/')), true);
        if (!is_array($claims) || ($claims['exp'] ?? 0) < time()) {
            throw new \RuntimeException('expired');
        }
        return $claims;
    }

    private function b64(string $data): string {
        return rtrim(strtr(base64_encode($data), '+/', '-_'), '=');
    }
}
```

## Comparação rápida

| Aspecto              | Sessão (cookie + store)             | JWT                                  |
| -------------------- | ----------------------------------- | ------------------------------------ |
| Estado               | Server-side                         | Client-side (no token)               |
| Logout               | Apaga store → instantâneo           | Difícil (precisa blacklist/short TTL)|
| Escala horizontal    | Precisa store compartilhado          | Trivial — só validar assinatura      |
| Tamanho da request   | Pequeno (cookie id)                 | Maior (token completo)               |
| Revogação            | Imediata                            | Exige blacklist ou rotação curta     |
| Multi-cliente        | Bom em web; ruim em mobile/3rd party| Ótimo                                |
| Ataque mais comum    | CSRF (mitigável SameSite)           | Vazamento de token, alg=none         |

## Quando usar / quando não usar

- **Sessão** — monolito server-rendered, app puramente web. Mais simples, mais seguro por padrão.
- **JWT** — APIs consumidas por múltiplos clientes (SPA + mobile + parceiro), microsserviços que precisam validar localmente. Ou OAuth2/OIDC (use lib pronta).
- **Híbrido** — refresh token de longa duração no servidor + access token JWT curto (≤15 min). É o padrão moderno.

## Armadilhas comuns

- `alg: none` aceito — falha histórica de libs. Sempre fixe os algoritmos esperados.
- JWT no localStorage — XSS rouba token. Use cookie HttpOnly + SameSite.
- TTL longo sem rotação — token vazado fica válido por horas. Refresh + curto TTL no access.
- Confiar em claims sem validar `iss`/`aud` — ataque de "confused deputy".

## Links relacionados

- [[PSRs]]
- [[Versionamento-Idempotencia]]
- [[Infra/Observabilidade-OWASP]]

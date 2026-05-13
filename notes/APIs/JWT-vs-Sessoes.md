---
tags: [php, api, auth, jwt, sessao]
fase: 5
status: draft
---

# JWT vs Sessões

## Conceito

Autenticação de requisições HTTP tem duas grandes famílias de implementação: **server-side sessions** (estado no servidor) e **tokens stateless** (estado no cliente). JWT é hoje o token stateless dominante, mas não o único — PASETO, Macaroons e Branca são alternativas com trade-offs diferentes.

### Sessão server-side

O servidor mantém o estado da sessão (usuário logado, roles, last activity, csrf token, carrinho) em um store: arquivo (PHP default), Memcached, Redis, banco SQL. O cliente recebe um cookie opaco — apenas o **session id** (geralmente 32 bytes aleatórios). Em cada requisição, o servidor faz lookup pelo id e carrega o estado.

PHP nativo: `session_start()`, dados em `$_SESSION`, persistidos em `/var/lib/php/sessions/sess_<id>` por default. Para produção, troca-se o handler para Redis (`ini_set('session.save_handler', 'redis')`) ou banco.

### JWT — JSON Web Token

Definido na RFC 7519 (2015). Estrutura: três blocos base64url separados por ponto.

- **Header**: `{"alg":"HS256","typ":"JWT"}` — algoritmo de assinatura e tipo.
- **Payload**: claims em JSON, divididas em registradas (padronizadas pela RFC) e privadas (definidas pela aplicação).
- **Signature**: HMAC ou assinatura assimétrica sobre `base64url(header) + "." + base64url(payload)`.

**Claims registradas** (RFC 7519 §4.1):
- `iss` (issuer) — quem emitiu o token.
- `sub` (subject) — a quem se refere (geralmente user id).
- `aud` (audience) — para qual serviço/API é destinado.
- `exp` (expiration time) — timestamp Unix de expiração.
- `nbf` (not before) — válido a partir de quando.
- `iat` (issued at) — quando foi emitido.
- `jti` (JWT id) — id único, útil para blacklist e replay protection.

**Algoritmos** (RFC 7518 — JWA):
- **HS256/HS384/HS512** — HMAC com chave simétrica. Mesmo segredo para assinar e verificar. Ótimo quando emissor e validador são o mesmo serviço.
- **RS256/RS384/RS512** — RSA assimétrica. Servidor de auth assina com chave privada; APIs validam com chave pública. Essencial em microsserviços (várias APIs validam tokens sem compartilhar segredo).
- **ES256** — ECDSA com curva P-256. Como RSA, porém chaves menores.
- **none** — **nunca aceite**. Vulnerabilidade histórica explorada em libs que aceitavam `alg: none`.

**JWS vs JWE**: JWT comum é JWS (assinado, payload legível). JWE é assinado + criptografado (payload opaco). Use JWE se o payload tiver dados sensíveis que não devem vazar via console do browser.

### Armazenamento do token no cliente

- **Cookie HttpOnly + Secure + SameSite=Lax/Strict** — recomendado. Não acessível via JavaScript (mitiga XSS roubando token). SameSite mitiga CSRF. Funciona automaticamente no browser.
- **localStorage / sessionStorage** — acessível via JS, vulnerável a XSS. Qualquer script injetado lê e exfiltra. Conveniente para SPAs que enviam token via `Authorization: Bearer ...`, mas exige higiene XSS extrema.
- **In-memory (variável JS)** — limpa em refresh; precisa silent re-auth. Mais seguro contra XSS persistente, mas UX pior.

### Revogação

Sessão: apaga linha do store. Logout é instantâneo e global.

JWT: token assinado é válido até `exp`. Para revogar antes:
- **Blacklist** — guardar `jti` revogados em Redis até `exp`. Quebra a stateless: cada validação consulta Redis.
- **TTL curto + refresh token** — access token vive 5-15min, refresh token (server-side) vive dias. Revogar = invalidar refresh; access expira sozinho rápido.
- **Versionamento de claims** — incluir `user_version` no token; bump no DB invalida todos tokens daquele usuário.

## Por que importa

A decisão é **arquitetural**, não preferência. JWT em monólito web tradicional é overkill com complicações de logout. Sessões em arquitetura distribuída multi-cliente (mobile + SPA + B2B + microsserviços) viram pesadelo de sticky-session ou exigem replicação de store entre regiões.

Cenário concreto 1 — **monólito Laravel + Blade**: tudo server-rendered, único cliente é o browser. Sessão Redis + cookie HttpOnly é mais simples, mais seguro por default, e logout funciona imediatamente. JWT aqui é overengineering.

Cenário concreto 2 — **SPA React + mobile iOS + mobile Android + API parceiro**: 4 tipos de cliente. Sessão exige cookie cross-domain (CORS complicado), mobile não tem cookies elegantes. JWT com Bearer header é uniforme em todos.

Cenário concreto 3 — **microsserviços com auth central**: serviço A emite token (RS256), serviços B/C/D validam localmente com chave pública. Sem JWT, cada serviço precisaria consultar serviço de auth a cada request — RTT extra de 10-50ms.

Cenário concreto 4 — **escala horizontal e regiões**: sessão exige Redis replicado entre regiões (latência cross-region, conflitos). JWT escala trivialmente — qualquer instância valida sem state compartilhado.

## Exemplo de código PHP

Exemplo 1 — emissor JWT HS256 (didático; em prod use firebase/php-jwt ou web-token/jwt-framework):

```php
<?php
declare(strict_types=1);

final readonly class Jwt
{
    public function __construct(
        private string $secret,
        private string $issuer = 'api.example.com',
        private string $audience = 'web',
    ) {}

    public function issue(int $userId, int $ttl = 900): string
    {
        $header  = $this->b64(json_encode(['alg' => 'HS256', 'typ' => 'JWT']));
        $payload = $this->b64(json_encode([
            'iss' => $this->issuer,
            'aud' => $this->audience,
            'sub' => (string) $userId,
            'iat' => time(),
            'nbf' => time(),
            'exp' => time() + $ttl,
            'jti' => bin2hex(random_bytes(8)),
        ]));
        $sig = $this->b64(hash_hmac('sha256', "{$header}.{$payload}", $this->secret, true));
        return "{$header}.{$payload}.{$sig}";
    }

    /** @return array<string, mixed> */
    public function verify(string $token): array
    {
        $parts = explode('.', $token, 3);
        if (count($parts) !== 3) {
            throw new \RuntimeException('malformed token');
        }
        [$h, $p, $s] = $parts;

        // SEMPRE valide algoritmo esperado — defesa contra alg: none
        $header = json_decode(base64_decode(strtr($h, '-_', '+/')), true);
        if (!is_array($header) || ($header['alg'] ?? null) !== 'HS256') {
            throw new \RuntimeException('unsupported alg');
        }

        $expected = $this->b64(hash_hmac('sha256', "{$h}.{$p}", $this->secret, true));
        if (!hash_equals($expected, $s)) {
            throw new \RuntimeException('invalid signature');
        }

        $claims = json_decode(base64_decode(strtr($p, '-_', '+/')), true);
        if (!is_array($claims)) {
            throw new \RuntimeException('invalid payload');
        }
        if (($claims['iss'] ?? null) !== $this->issuer) {
            throw new \RuntimeException('issuer mismatch');
        }
        if (($claims['aud'] ?? null) !== $this->audience) {
            throw new \RuntimeException('audience mismatch');
        }
        if (($claims['exp'] ?? 0) < time()) {
            throw new \RuntimeException('expired');
        }
        if (($claims['nbf'] ?? 0) > time()) {
            throw new \RuntimeException('not yet valid');
        }
        return $claims;
    }

    private function b64(string $data): string
    {
        return rtrim(strtr(base64_encode($data), '+/', '-_'), '=');
    }
}
```

Exemplo 2 — refresh token rotation com Redis:

```php
<?php
declare(strict_types=1);

final readonly class TokenPair
{
    public function __construct(
        public string $accessToken,
        public string $refreshToken,
    ) {}
}

final readonly class AuthService
{
    public function __construct(
        private Jwt $jwt,
        private \Redis $redis,
    ) {}

    public function login(int $userId): TokenPair
    {
        $access  = $this->jwt->issue($userId, ttl: 900);          // 15 min
        $refresh = bin2hex(random_bytes(32));
        $this->redis->setex("refresh:{$refresh}", 60 * 60 * 24 * 14, (string) $userId); // 14 dias
        return new TokenPair($access, $refresh);
    }

    public function refresh(string $oldRefresh): TokenPair
    {
        $userId = $this->redis->get("refresh:{$oldRefresh}");
        if ($userId === false) {
            throw new \RuntimeException('invalid refresh token');
        }
        // rotation: invalida o antigo, emite novo par
        $this->redis->del("refresh:{$oldRefresh}");
        return $this->login((int) $userId);
    }

    public function logout(string $refresh): void
    {
        $this->redis->del("refresh:{$refresh}");
        // access token continua valido até exp natural (15 min)
    }
}
```

Exemplo 3 — middleware de auth com cookie HttpOnly:

```php
<?php
declare(strict_types=1);

final readonly class JwtAuthMiddleware implements MiddlewareInterface
{
    public function __construct(private Jwt $jwt) {}

    public function process(
        ServerRequestInterface $req,
        RequestHandlerInterface $next,
    ): ResponseInterface {
        // 1) prioritariamente cookie HttpOnly (browser)
        $token = $req->getCookieParams()['access_token'] ?? null;

        // 2) fallback Authorization: Bearer (mobile/server)
        if ($token === null) {
            $auth = $req->getHeaderLine('Authorization');
            if (str_starts_with($auth, 'Bearer ')) {
                $token = substr($auth, 7);
            }
        }

        if ($token === null) {
            return new Response(401);
        }
        try {
            $claims = $this->jwt->verify($token);
        } catch (\RuntimeException) {
            return new Response(401);
        }
        return $next->handle($req->withAttribute('user_id', (int) $claims['sub']));
    }
}
```

Exemplo 4 — sessão PHP via Redis (alternativa stateful):

```php
<?php
declare(strict_types=1);

ini_set('session.save_handler', 'redis');
ini_set('session.save_path', 'tcp://redis:6379?auth=secret');
ini_set('session.cookie_httponly', '1');
ini_set('session.cookie_secure', '1');
ini_set('session.cookie_samesite', 'Lax');
ini_set('session.use_strict_mode', '1');
ini_set('session.gc_maxlifetime', '3600');

session_start();

if (!isset($_SESSION['user_id'])) {
    http_response_code(401);
    exit;
}
$userId = $_SESSION['user_id'];
```

## Comparação

| Aspecto | Sessão (cookie + store) | JWT |
| --- | --- | --- |
| Estado | Server-side | Client-side (no token) |
| Logout | Apaga store: instantâneo | Difícil (blacklist ou TTL curto) |
| Escala horizontal | Precisa store compartilhado | Trivial: só validar assinatura |
| Latência por request | Lookup (Redis ~1ms) | CPU (HMAC ~0.1ms; RSA ~1ms) |
| Tamanho da request | Pequeno (~32 bytes id) | Médio (200-1000 bytes) |
| Revogação | Imediata | Atrasada ou stateful |
| Multi-cliente | Bom em web; ruim em mobile/3rd party | Ótimo |
| Cross-domain | Difícil (cookies + CORS) | Fácil (Authorization header) |
| XSS | Cookie HttpOnly mitiga | localStorage vulnerável; cookie ok |
| CSRF | Risco se cookies não SameSite | Sem risco se header-based |
| Trafegabilidade | Cookie automático | Manual (a cada request) |
| Federação (SSO/OIDC) | Pobre | Nativa |

## Quando usar / quando não usar

- **Sessão server-side** — monólito server-rendered, app puramente web do mesmo domínio. Mais simples, mais seguro por default.
- **JWT** — APIs consumidas por múltiplos clientes (SPA + mobile + parceiro), microsserviços que validam localmente, OAuth2/OIDC, federação SSO.
- **Híbrido (recomendado para SPA + API)** — refresh token de longa duração no servidor (stateful, revogável) + access token JWT curto (≤15 min, stateless). Combina melhor dos dois mundos.
- **PASETO** — quando quiser segurança por default sem ter de escolher algoritmo (v4 padroniza Ed25519). Bom em greenfield.
- **Não use JWT puro** em app monolítico sem necessidade de escala stateless — você está pagando complexidade sem ganho.
- **Não use sessão** quando token precisa atravessar domínios não-controlados (B2B com parceiros).

## Armadilhas comuns

- **`alg: none` aceito**: falha histórica de libs (e bugs em validadores ingênuos). Sempre **fixe** o algoritmo esperado no código de verificação. Nunca confie no `alg` do header.
- **Confusão HS256 vs RS256**: passar chave pública RSA como segredo HMAC. Atacante assina token com a "chave pública" como HMAC secret e burla. Use libs que validam tipo de chave x algoritmo.
- **JWT em localStorage**: XSS rouba token e ataca por dias até `exp`. Use cookie HttpOnly + Secure + SameSite. Se precisar de SPA cross-origin, considere BFF pattern (cookie no BFF, JWT só backend-to-backend).
- **TTL longo sem rotação**: token vazado fica válido por horas. Combine TTL curto (5-15min) com refresh token rotation.
- **Não validar `iss` e `aud`**: ataque "confused deputy" — token emitido para serviço X é aceito por serviço Y. Sempre valide audience.
- **Payload com dados sensíveis**: JWT é base64, não criptografado. Email, telefone, role detalhado vazam em logs/proxies. Use JWE ou só `sub` opaco.
- **Refresh token sem rotation**: vazou uma vez, atacante usa para sempre. Cada refresh emite par novo e invalida o anterior.
- **Logout que não invalida refresh**: cliente "saiu", mas refresh ainda emite novos access tokens. Apague o refresh do store ao deslogar.
- **Replay attack sem `jti`**: copiar e reenviar token. Em APIs sensíveis, mantenha cache de `jti` usados (especialmente em one-time-use tokens).
- **Clock skew entre servidores**: `exp` rejeita token por 2s de diferença NTP. Permita leeway de 30-60s na validação.

## Links relacionados

- [[PSRs]]
- [[REST-Maduro]]
- [[Versionamento-Idempotencia]]
- [[OpenAPI]]
- [[Infra/Observabilidade-OWASP]]

---
tags: [php, infra, observabilidade, otel, owasp, seguranca]
fase: 7
status: stub
---

# Observabilidade + OWASP Top 10

## Conceito

**Observabilidade**: capacidade de entender o estado interno do sistema só pelos sinais externos. Três pilares:

- **Logs** — eventos discretos, idealmente JSON estruturado com `trace_id`.
- **Metrics** — séries temporais (RPS, latência, erros).
- **Traces** (OpenTelemetry) — rastros de uma request atravessando vários serviços.

**OWASP Top 10** — lista das 10 vulnerabilidades web mais comuns. Em PHP, as mais frequentes: A01 Broken Access Control, A02 Cryptographic Failures, A03 Injection (SQLi, XSS), A05 Security Misconfiguration.

## Por que importa

Sistema sem observabilidade é caixa-preta — incidente vira interrogatório. OWASP Top 10 não é trivia: cada item é vetor de ataque ativo em produção todo dia.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// === log estruturado JSON com correlação ===
use Monolog\Logger;
use Monolog\Handler\StreamHandler;
use Monolog\Formatter\JsonFormatter;

final class StructuredLogger {
    public static function build(): Logger {
        $handler = new StreamHandler('php://stdout');
        $handler->setFormatter(new JsonFormatter());

        $logger = new Logger('app');
        $logger->pushHandler($handler);
        // processor que adiciona trace_id de contexto request
        $logger->pushProcessor(function (array $record): array {
            $record['extra']['trace_id'] = TraceContext::current()?->traceId;
            $record['extra']['service']  = 'orders-api';
            $record['extra']['version']  = getenv('APP_VERSION');
            return $record;
        });
        return $logger;
    }
}

// === proteção contra SQLi: SEMPRE parametrizado ===
// ❌ vulnerável
$pdo->query("SELECT * FROM orders WHERE id = {$_GET['id']}");

// ✅ correto
$stmt = $pdo->prepare('SELECT * FROM orders WHERE id = ?');
$stmt->execute([$_GET['id']]);

// === proteção XSS: escape no momento da saída ===
echo htmlspecialchars($userInput, ENT_QUOTES | ENT_HTML5, 'UTF-8');

// === segredos: nunca em log ===
$logger->info('payment.attempted', [
    'order_id' => $id->toString(),
    // 'card_number' => $card,  ← NUNCA
    'last4' => substr($card, -4),
]);
```

## OWASP Top 10 — cheatsheet PHP

| # | Item                          | Mitigação em PHP                                              |
| - | ----------------------------- | ------------------------------------------------------------- |
| 1 | Broken Access Control         | Autorização por use case; deny-by-default; testes negativos   |
| 2 | Cryptographic Failures        | `password_hash` (bcrypt/argon2id); TLS 1.2+; segredos no vault|
| 3 | Injection                     | PDO prepared, escape no output, validar/aceitar VOs           |
| 4 | Insecure Design               | Threat modeling; menor privilégio                             |
| 5 | Security Misconfiguration     | `expose_php=Off`, `display_errors=Off`, headers de segurança  |
| 6 | Vulnerable Components         | `composer audit` no CI; Renovate/Dependabot                   |
| 7 | Identification & Auth Failures| Lockout, MFA, sessão segura, ver [[APIs/JWT-vs-Sessoes]]      |
| 8 | Software/Data Integrity       | Signed releases; SBOM; subresource integrity                  |
| 9 | Security Logging Failures     | Logs estruturados, alertas, retenção                          |
|10 | SSRF                          | Allowlist de hosts, validar URLs antes de fetch               |

## Headers de segurança (response)

```php
$res = $res
    ->withHeader('X-Content-Type-Options', 'nosniff')
    ->withHeader('X-Frame-Options', 'DENY')
    ->withHeader('Referrer-Policy', 'no-referrer')
    ->withHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains')
    ->withHeader('Content-Security-Policy', "default-src 'self'");
```

## Quando usar / quando não usar

- **Logs JSON + trace_id** — sempre em qualquer serviço de produção.
- **OpenTelemetry** — assim que houver ≥2 serviços comunicando. Em monolito, métricas + logs já cobrem.
- **`composer audit`** no CI sempre.
- **Não** depender só de WAF para proteção. WAF é camada extra, não substituto de código seguro.

## Armadilhas comuns

- Logar payload inteiro (cartão, senha, JWT). Sanitize antes.
- `display_errors=On` em produção mostra senha de banco no stack trace.
- Logger síncrono que bloqueia em I/O (disco lento) — derruba latência. Use buffered/async.
- Métricas sem cardinalidade controlada — explode em Prometheus (label `user_id` em métrica gerou caso clássico de OOM).

## Links relacionados

- [[Fundamentos/Hierarquia-Throwable]]
- [[Fundamentos/Configuracao-PHP-INI]]
- [[APIs/JWT-vs-Sessoes]]
- [[CI-CD]]

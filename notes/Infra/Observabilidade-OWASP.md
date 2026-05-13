---
tags: [php, infra, observabilidade, otel, owasp, seguranca]
fase: 7
status: draft
---

# Observabilidade + OWASP Top 10

## Conceito

**Observabilidade** é a capacidade de inferir o estado interno de um sistema unicamente pelos sinais que ele emite. É diferente de monitoramento: monitoramento responde "o sistema está OK?" com base em checks pré-definidos; observabilidade responde "**por que** o sistema não está OK?" mesmo diante de comportamento nunca visto antes. A diferença prática aparece em incidente novo — monitoramento mostra alerta, observabilidade conta a história.

Os **três pilares** complementam-se: **logs** são eventos discretos com contexto (uma linha por acontecimento); **métricas** são séries temporais agregadas (contadores, gauges, histogramas amostrados em intervalos); **traces** são grafos causais de uma requisição atravessando serviços (spans aninhados ligados por `trace_id`). Em sistema bem instrumentado, partir de um alerta em métrica → drillar para traces da janela → abrir logs do span suspeito é fluxo de 30 segundos.

O **modelo mental** para logs é "evento estruturado, não string para humano". `$logger->info('user logged in: ' . $email)` é grep-bait — funciona para uma máquina, quebra em escala. `$logger->info('user.login', ['email' => $email, 'ip' => $ip, 'request_id' => $rid])` é consultável: filtre por `event=user.login AND ip=...`, agregue, alerte. Logs estruturados em JSON são o padrão atual; cada linha é um objeto auto-contido com timestamp, severidade, evento, contexto e identificadores de correlação.

**Correlation ID** (também chamado `request_id` ou `trace_id`) é o tendão que liga as três dimensões. Um middleware na entrada da requisição gera (ou propaga via header `X-Request-Id` / `traceparent`) um UUID, injeta em todo logger contextual, e propaga em chamadas downstream. Numa investigação, esse ID é a chave que conecta o erro 500 que o cliente reportou, a métrica que pulou, o trace que mostrou query lenta, e o log de exceção que veio do worker.

**OpenTelemetry (OTel)** unificou o ecossistema de instrumentação. É especificação aberta + SDKs (PHP tem extensão oficial `ext-opentelemetry` + composer packages) + protocolo OTLP. Você instrumenta uma vez (manual ou via autoinstrumentação) e exporta para qualquer backend compatível: Jaeger, Tempo, Datadog, NewRelic, Honeycomb, Grafana Cloud. Acabou o lock-in de SDK proprietário.

**OWASP Top 10** é o consenso decenal da indústria sobre as classes de vulnerabilidade web mais comuns. A edição 2021 (vigente até nova revisão) lista A01-A10. Cada item é vetor de ataque ativo em produção todo dia — não é trivia acadêmica. Em PHP especificamente, os mais frequentes são A01 (Broken Access Control), A03 (Injection — SQLi, XSS), A05 (Security Misconfiguration) e A06 (Vulnerable Components).

## Por que importa

**Sistema sem observabilidade é caixa-preta em incidente**. Cliente reporta "lento desde 14h"; ninguém sabe se foi deploy, banco, dependência externa, picos de tráfego ou bug latente. A investigação vira interrogatório: pergunta-se "o que mudou?" e ninguém tem resposta com tempo. Com observabilidade adequada, a janela do alerta dá histograma de latência, traces da janela apontam o span que dobrou de latência, logs do span mostram tipo de query, e em 5 minutos você sabe se rola rollback, hotfix ou comunicação ao cliente.

**OWASP não é teoria**. Cada item da lista resulta em vazamento de dados, multa LGPD/GDPR, ransomware, defacement, fraude. A01 (Broken Access Control) é a #1 estatisticamente: APIs que checam autenticação mas esquecem autorização ("você está logado, então pode acessar tudo"). A03 (Injection) continua ativa apesar de prepared statements existirem há 20 anos — sempre que tem `query("... $variavel ...")` no codebase, há risco.

**Cenário 1 — Incidente sem traces**: API REST com 4 microsserviços. Cliente reporta timeout em endpoint. Sem traces, você abre logs de cada serviço, tenta correlacionar por timestamp, descobre que serviço B chamou C que chamou D que chamou um banco em outra região. Com traces, abre o span, vê 1.2s em chamada externa, fim.

**Cenário 2 — SSRF via image proxy**: feature inocente "carregue imagem da URL X para o avatar". Sem allowlist de hosts, atacante pede `http://169.254.169.254/latest/meta-data/iam/security-credentials/` (metadata do EC2/GCP), serviço busca, devolve credenciais. Vazamento total da conta cloud. Mitigação: allowlist + bloqueio explícito de IPs privados e link-local.

**Cenário 3 — Composer audit ignorado**: equipe adia atualização do `guzzlehttp/guzzle` por 6 meses. Vulnerabilidade crítica de SSRF anunciada. Sem `composer audit` no CI, time descobre quando o Shodan já mapeou. Com audit em PR, CI marca vermelho no dia 1 e bump é trivial.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// === Logger estruturado JSON com correlation ID ===
use Monolog\Logger;
use Monolog\Level;
use Monolog\Handler\StreamHandler;
use Monolog\Handler\RotatingFileHandler;
use Monolog\Formatter\JsonFormatter;
use Monolog\Processor\PsrLogMessageProcessor;
use Monolog\Processor\IntrospectionProcessor;

final class LoggerFactory
{
    public static function build(string $service): Logger
    {
        $logger = new Logger($service);

        // STDOUT → coletor (Fluent Bit, Vector, Promtail) lê dali
        $stdout = new StreamHandler('php://stdout', Level::Info);
        $stdout->setFormatter(new JsonFormatter(
            batchMode: JsonFormatter::BATCH_MODE_NEWLINES,
            appendNewline: true,
            ignoreEmptyContextAndExtra: true,
        ));
        $logger->pushHandler($stdout);

        // Arquivo separado para erros (retenção maior)
        $errorFile = new RotatingFileHandler(
            filename: '/var/log/app/error.log',
            maxFiles: 30,
            level: Level::Error,
        );
        $errorFile->setFormatter(new JsonFormatter());
        $logger->pushHandler($errorFile);

        // Processadores que enriquecem todos os records
        $logger->pushProcessor(new PsrLogMessageProcessor());
        $logger->pushProcessor(new IntrospectionProcessor(Level::Warning));
        $logger->pushProcessor(static function (array $record): array {
            $record['extra']['request_id'] = RequestContext::id() ?? 'no-request';
            $record['extra']['trace_id']   = TraceContext::current()?->traceId;
            $record['extra']['span_id']    = TraceContext::current()?->spanId;
            $record['extra']['service']    = getenv('APP_NAME') ?: 'unknown';
            $record['extra']['version']    = getenv('APP_VERSION') ?: 'dev';
            $record['extra']['env']        = getenv('APP_ENV') ?: 'local';
            return $record;
        });

        return $logger;
    }
}

// Uso típico — sempre evento + contexto, nunca interpolação manual
$logger->info('payment.attempted', [
    'order_id' => $order->id->toString(),
    'amount'   => $order->total->amount,
    'currency' => $order->total->currency,
    'user_id'  => $user->id,
    // NUNCA: 'card_number' => $card
    'last4'    => substr($card, -4),
]);
```

```php
<?php
// === OpenTelemetry: trace manual de um caso de uso ===
use OpenTelemetry\API\Globals;
use OpenTelemetry\API\Trace\StatusCode;
use OpenTelemetry\API\Trace\SpanKind;

final class PlaceOrderHandler
{
    public function __invoke(PlaceOrderCommand $cmd): OrderId
    {
        $tracer = Globals::tracerProvider()->getTracer('orders');

        $span = $tracer->spanBuilder('place_order')
            ->setSpanKind(SpanKind::KIND_INTERNAL)
            ->setAttribute('order.items_count', count($cmd->items))
            ->setAttribute('user.id', $cmd->userId)
            ->startSpan();

        $scope = $span->activate();
        try {
            $order = $this->orders->create($cmd);
            $this->payments->charge($order);
            $this->events->publish(new OrderPlaced($order->id));

            $span->setAttribute('order.id', $order->id->toString());
            $span->setStatus(StatusCode::STATUS_OK);
            return $order->id;
        } catch (Throwable $e) {
            $span->recordException($e);
            $span->setStatus(StatusCode::STATUS_ERROR, $e->getMessage());
            throw $e;
        } finally {
            $scope->detach();
            $span->end();
        }
    }
}
```

```php
<?php
// === Métricas Prometheus via promphp/prometheus_client_php ===
use Prometheus\CollectorRegistry;
use Prometheus\Storage\Redis as RedisStorage;

$registry = new CollectorRegistry(new RedisStorage());

$httpRequests = $registry->getOrRegisterCounter(
    namespace: 'app',
    name: 'http_requests_total',
    help: 'Total de requisições HTTP',
    labels: ['method', 'route', 'status'],   // CARDINALIDADE: cuidado
);

$httpDuration = $registry->getOrRegisterHistogram(
    namespace: 'app',
    name: 'http_request_duration_seconds',
    help: 'Latência das requisições',
    labels: ['method', 'route'],
    buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
);

// Middleware
$start = hrtime(true);
$response = $next($request);
$elapsed = (hrtime(true) - $start) / 1e9;

$httpRequests->inc([$request->getMethod(), $route->name, (string)$response->getStatusCode()]);
$httpDuration->observe($elapsed, [$request->getMethod(), $route->name]);
```

```php
<?php
// === OWASP A03 Injection — prepared statements SEMPRE ===

// VULNERÁVEL: SQLi clássica
$pdo->query("SELECT * FROM orders WHERE id = {$_GET['id']}");

// CORRETO: parametrização
$stmt = $pdo->prepare('SELECT * FROM orders WHERE id = :id');
$stmt->execute(['id' => $_GET['id']]);
$order = $stmt->fetch(PDO::FETCH_ASSOC);

// XSS: escape no MOMENTO DA SAÍDA, contextual
echo htmlspecialchars($userInput, ENT_QUOTES | ENT_HTML5, 'UTF-8');

// Em template engines, escape automático (Twig, Blade)
// Em context JS dentro de HTML: json_encode com flags de segurança
echo '<script>var data = '
   . json_encode($data, JSON_HEX_TAG | JSON_HEX_AMP | JSON_HEX_APOS | JSON_HEX_QUOT | JSON_THROW_ON_ERROR)
   . ';</script>';
```

```php
<?php
// === OWASP A10 SSRF — allowlist + bloqueio de IPs internos ===
final class SafeHttpFetcher
{
    private const ALLOWED_HOSTS = ['cdn.example.com', 'images.partner.com'];

    public function fetch(string $url): string
    {
        $parts = parse_url($url);
        if ($parts === false || !isset($parts['host'], $parts['scheme'])) {
            throw new InvalidArgumentException('URL malformada');
        }
        if (!in_array($parts['scheme'], ['http', 'https'], strict: true)) {
            throw new InvalidArgumentException('Esquema proibido');
        }
        if (!in_array($parts['host'], self::ALLOWED_HOSTS, strict: true)) {
            throw new InvalidArgumentException('Host não permitido');
        }

        // Resolver DNS e bloquear faixas privadas (proteção contra DNS rebinding)
        $ip = gethostbyname($parts['host']);
        if ($this->isPrivateOrLoopback($ip)) {
            throw new RuntimeException('IP interno bloqueado');
        }

        return file_get_contents($url);
    }

    private function isPrivateOrLoopback(string $ip): bool
    {
        return !filter_var(
            $ip,
            FILTER_VALIDATE_IP,
            FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE,
        );
    }
}
```

```php
<?php
// === Headers de segurança (response middleware) ===
$response = $response
    ->withHeader('X-Content-Type-Options', 'nosniff')
    ->withHeader('X-Frame-Options', 'DENY')
    ->withHeader('Referrer-Policy', 'no-referrer')
    ->withHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains; preload')
    ->withHeader('Content-Security-Policy',
        "default-src 'self'; "
        . "script-src 'self' 'nonce-{$nonce}'; "
        . "style-src 'self' 'unsafe-inline'; "
        . "img-src 'self' data: https:; "
        . "frame-ancestors 'none'; "
        . "base-uri 'self'; "
        . "form-action 'self'")
    ->withHeader('Permissions-Policy', 'geolocation=(), camera=(), microphone=()');
```

## OWASP Top 10 (2021) — cheatsheet PHP

| #   | Item                              | Como aparece em PHP                                      | Mitigação                                                                                  |
| --- | --------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| A01 | Broken Access Control             | Endpoints que só checam auth, esquecem authz             | Policy/voter por caso de uso; deny-by-default; testes negativos no CI                      |
| A02 | Cryptographic Failures            | `md5`/`sha1` para senha; HTTP sem HSTS; segredos hardcoded | `password_hash` (argon2id/bcrypt); TLS 1.2+; HSTS; segredos em vault (Vault, AWS SM)         |
| A03 | Injection                         | SQLi por concatenação; XSS por `echo $userInput`         | PDO prepared statements; `htmlspecialchars` no output; Twig/Blade auto-escape; CSP         |
| A04 | Insecure Design                   | Recuperação de senha por pergunta secreta; sem rate limit | Threat modeling; design review; rate limit por IP e por user                              |
| A05 | Security Misconfiguration         | `display_errors=On`; `expose_php=On`; debugbar em prod    | INI hardening; `composer install --no-dev`; remover arquivos `.env`, `phpinfo.php`         |
| A06 | Vulnerable and Outdated Components | Lib com CVE nunca atualizada                              | `composer audit` no CI; Renovate/Dependabot; SCA tools (Snyk)                              |
| A07 | Identification & Auth Failures    | Senhas fracas; sem lockout; sessão sem rotação            | Política de senha; rate limit + lockout; rotacionar `session_id` no login; MFA; ver [[APIs/JWT-vs-Sessoes]] |
| A08 | Software & Data Integrity Failures | Update via HTTP; sem verificação de signature do release | Signed releases; SBOM; subresource integrity; `composer.lock` versionado                   |
| A09 | Security Logging & Monitoring Failures | Não loga falha de login; alerta sem destinatário     | Logs estruturados de eventos de segurança; SIEM; alertas com runbook                       |
| A10 | Server-Side Request Forgery (SSRF) | Image proxy sem allowlist; webhook URL configurável       | Allowlist de hosts; bloqueio de IPs privados/link-local; rede separada (egress controlado) |

## Stack típica de observabilidade

| Camada    | Open source                          | Comercial                       |
| --------- | ------------------------------------ | ------------------------------- |
| Coleta    | Fluent Bit, Vector, OTel Collector   | Datadog Agent, NewRelic Agent   |
| Logs      | Loki + Grafana, Elastic (ELK)        | Datadog Logs, Splunk            |
| Métricas  | Prometheus + Grafana                 | Datadog Metrics, NewRelic       |
| Traces    | Jaeger, Tempo                        | Datadog APM, NewRelic, Honeycomb |
| Alertas   | Alertmanager, Grafana OnCall         | PagerDuty, Opsgenie             |
| SAST      | PHPStan, Psalm, Semgrep, Phan        | Snyk Code, SonarQube            |
| SCA       | `composer audit`, Dependabot         | Snyk, Mend                      |
| DAST      | OWASP ZAP, Nikto                     | Burp Suite Pro                  |

## Quando usar / quando não usar

- **Logs JSON estruturados + correlation ID**: sempre em qualquer serviço de produção. Custo zero, retorno imediato.
- **OpenTelemetry**: assim que houver dois ou mais serviços conversando. Em monolito isolado, métricas + logs cobrem.
- **Métricas Prometheus**: sempre. Mínimo: contador de requests por status, histograma de latência, contador de erros por tipo.
- **APM comercial (Datadog, NewRelic)**: quando o time não tem capacidade para operar stack open source ou o volume justifica. Excelente time-to-value, custo crescente.
- **`composer audit` em CI**: sempre. Custo: 10 segundos por build.
- **WAF (Cloudflare, AWS WAF)**: camada extra, **nunca substituto** de código seguro. WAF filtra padrões conhecidos, código seguro filtra os desconhecidos.
- **CSP rigorosa**: alvo de longo prazo. Comece em `Content-Security-Policy-Report-Only` e endureça.
- **Não use** logs como banco de dados (queries complexas em volume alto custam fortuna).
- **Não use** label de alta cardinalidade em métricas (`user_id`, `request_id` em Prometheus quebra o storage).

## Armadilhas comuns

- **Logar payload sensível inteiro**: cartão, CVV, senha, JWT, refresh token, CPF/CNPJ, endereço. Vira vazamento permanente porque log é replicado, indexado, backup-ado. Sanitize antes do log; mantenha denylist de campos.
- **`display_errors=On` em produção**: stack trace expõe senha de banco em string de conexão, paths absolutos, versões de libs. Sempre `Off` em prod, com log para arquivo + sistema de erro centralizado (Sentry).
- **Logger síncrono bloqueante em I/O lento**: disco lento ou socket de log saturado segura o request thread. Use handlers buffered/async; ou direcione para STDOUT e deixe o coletor lidar.
- **Cardinalidade explosiva em métricas**: label com `user_id` em Prometheus cria uma série temporal por usuário; 100k usuários = 100k séries por métrica = OOM no Prometheus. Use só labels de baixa cardinalidade (`method`, `route`, `status`).
- **Sem rotação de `session_id` no login**: ataque de session fixation — atacante força vítima a usar sessão dele, depois login da vítima dá acesso. Sempre `session_regenerate_id(true)` no login e logout.
- **Trust em headers `X-Forwarded-For` sem validar proxy**: atacante manda `X-Forwarded-For: 127.0.0.1` direto e passa por filtro de IP. Configure trusted proxies explicitamente.
- **CORS com `Access-Control-Allow-Origin: *` em endpoint autenticado**: permite que qualquer site leia respostas da API com cookies do usuário. Use allowlist de origens.
- **Sem `composer audit` no CI**: descobre CVE 6 meses depois. Adicione ao pipeline.
- **`md5`/`sha1` para senha**: ambos quebrados há mais de uma década. Use `password_hash($pwd, PASSWORD_ARGON2ID)` e `password_verify`.
- **JWT sem validação de `alg`**: aceita `alg: none` ou troca `HS256` por `RS256` com chave pública atacante. Sempre valide algoritmo explicitamente. Ver [[APIs/JWT-vs-Sessoes]].
- **Upload de arquivo sem validar tipo real**: confiar em `$_FILES['x']['type']` (header do cliente) é falha; valide com `finfo_file($path, FILEINFO_MIME_TYPE)` e renomeie para extensão segura. Sirva uploads de domínio separado.
- **SSRF em features de "fetch URL"**: image proxy, webhooks configuráveis, importadores. Sempre allowlist de hosts e bloqueio de IPs internos (169.254.169.254, 10.x, 192.168.x, 127.x).

## Links relacionados

- [[Fundamentos/Hierarquia-Throwable]]
- [[Fundamentos/Configuracao-PHP-INI]]
- [[APIs/JWT-vs-Sessoes]]
- [[APIs/CORS-Rate-Limit-Idempotencia]]
- [[CI-CD]]
- [[Docker-FPM]]
- [[Performance/Workers-Filas-Async]]

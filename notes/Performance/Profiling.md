---
tags: [php, performance, profiling]
fase: 6
status: draft
---

# Profiling: Xdebug, Blackfire, SPX

## Conceito

**Profiling** é a coleta sistemática de dados sobre onde um programa gasta tempo e memória durante a execução. Difere de **benchmarking** (medir a duração total de uma operação fechada) e de **tracing** distribuído (acompanhar uma request através de múltiplos serviços). Profiling responde à pergunta "dentro deste processo, qual função consome 40% do wall-clock?".

Existem duas grandes famílias de profilers, com trade-offs profundos. **Instrumentação** insere ganchos em cada entrada/saída de função — produz dados exatos (contagem de chamadas, tempo inclusivo/exclusivo por função), porém adiciona overhead significativo (Xdebug profile mode causa 2-10x slowdown) que pode até distorcer o resultado (efeito observador: I/O proporcionalmente diminui, função CPU pesada parece mais cara). **Sampling** interrompe o processo em intervalos regulares (ex.: 100 Hz) e captura a stack — overhead baixo (≤5%), mas funções rápidas podem ser perdidas estatisticamente. Blackfire e SPX combinam ambas as abordagens; profilers como php-spy e Datadog Continuous Profiler são sampling puros.

O ecossistema PHP dispõe de quatro ferramentas principais. **Xdebug** (Derick Rethans) é o canivete suíço de debug: step-debugger, var_dump melhorado, trace, profile (formato Cachegrind). É instrumentação pura — útil para análise pontual em dev, **proibido em produção**. **Blackfire.io** (SensioLabs) é comercial, projetada para rodar em produção com overhead controlado, oferece comparações entre branches (CI integration), recomendações automáticas ("você fez 47 queries idênticas, considere cache"), e call-graphs interativos. **SPX** (NoiseByNorthwest) é open-source, leve, com UI web embutida — excelente alternativa gratuita ao Blackfire para staging. **Tideways** é comercial, foco em APM contínuo (monitoramento perene de produção), similar ao New Relic mas especializado em PHP.

Além desses, ferramentas APM gerais — **New Relic**, **Datadog**, **Elastic APM**, **Scout** — fazem profiling agregado por endpoint, mostrando p50/p95/p99, breakdown de tempo por categoria (DB, cache, HTTP externo, code), e flame graphs. Não substituem um profiler dedicado para investigação pontual, mas são imprescindíveis para *saber qual endpoint vale profilear*.

A disciplina fundamental: **otimização orientada a profile**. A "Lei de Knuth" — *premature optimization is the root of all evil* — significa exatamente isso: não otimize sem medir. 80% do tempo gasto em micro-otimizações sem profile vai para o lugar errado. Existe um conceito chave: **hot path** (caminho crítico, atravessado em quase toda request — vale otimizar tudo aqui) versus **cold path** (raramente atingido — otimizar é desperdício, foco deve ser clareza). Profile revela qual é qual.

## Por que importa

Cenário real: um e-commerce reportava `POST /checkout` com p95 de 4.2s. Sem profile, suspeitas circulavam: "deve ser o gateway de pagamento", "Doctrine está lento", "Redis travando". Um profile de 5 minutos com Blackfire mostrou:

| Categoria                     | Tempo (% do total) |
| ----------------------------- | ------------------ |
| `json_decode` em loop         | 38%                |
| Consultas N+1 ao banco        | 24%                |
| Gateway de pagamento (HTTP)   | 19%                |
| Rendering Twig                | 8%                 |
| Resto (logs, eventos, etc.)   | 11%                |

A causa raiz era um `json_decode` chamado dentro de loop sobre 200 produtos do carrinho, parseando o mesmo blob de 80KB toda iteração. Correção: parse uma vez, cache em variável local. Resultado: p95 caiu de 4.2s para 1.1s — sem tocar no gateway nem no ORM. **Sem profile, esse fix nunca seria encontrado.**

Outra métrica importante: profile dirige a decisão de **escala vertical vs horizontal**. Se 70% do tempo é CPU em código aplicacional, escalar horizontalmente ajuda. Se 70% é espera por banco (`pdo_mysql` em wait), adicionar servidores PHP só esquenta o pool de conexões — gargalo está em outro lugar.

## Exemplo de código PHP

```ini
; Xdebug profile mode (PHP 8.x, Xdebug 3.x)
; Em dev, com trigger explícito
xdebug.mode               = profile,debug
xdebug.output_dir         = /tmp/xdebug
xdebug.profiler_output_name = cachegrind.out.%t.%p
xdebug.start_with_request = trigger
; Ativar via ?XDEBUG_PROFILE=1 ou cookie XDEBUG_PROFILE
; Analisar: KCachegrind / QCachegrind / Webgrind
```

```ini
; SPX — profiler open-source com UI web
spx.http_enabled         = 1
spx.http_key             = "PALAVRA-SECRETA-LONGA"
spx.http_ip_whitelist    = "127.0.0.1,10.0.0.0/8"
spx.data_dir             = /var/lib/spx
; UI em http://app.local/?SPX_UI_URI=/&SPX_KEY=PALAVRA-SECRETA-LONGA
; Ativar profile: cookie SPX_ENABLED=1 ou header SPX-Enabled
```

```ini
; Blackfire — agent + extensão
blackfire.agent_socket    = tcp://blackfire-agent:8307
blackfire.server_id       = ${BLACKFIRE_SERVER_ID}
blackfire.server_token    = ${BLACKFIRE_SERVER_TOKEN}
; Profile via CLI: blackfire run php script.php
; Profile via browser: extensão Blackfire Companion
; Profile via CI: blackfire run --reference=prod phpunit
```

```php
<?php
declare(strict_types=1);

// Micro-benchmark manual — para validar otimização local
// (não substitui profile real; serve para confirmar hipóteses)
final class Bench
{
    public static function run(string $label, callable $fn, int $iters = 100_000): void
    {
        // warm-up (caches CPU, OPcache JIT)
        for ($i = 0; $i < 1000; $i++) {
            $fn();
        }

        $memBefore = memory_get_usage();
        $t0 = hrtime(true);

        for ($i = 0; $i < $iters; $i++) {
            $fn();
        }

        $elapsedNs = hrtime(true) - $t0;
        $memDelta  = memory_get_usage() - $memBefore;

        printf(
            "%-35s %8.3f ms total | %8.2f ns/op | %+d B mem\n",
            $label,
            $elapsedNs / 1e6,
            $elapsedNs / $iters,
            $memDelta,
        );
    }
}

Bench::run('array_map + arrow',  fn () => array_map(fn ($x) => $x * 2, range(1, 100)));
Bench::run('foreach manual',     function () {
    $out = [];
    foreach (range(1, 100) as $x) {
        $out[] = $x * 2;
    }
    return $out;
});
Bench::run('json_decode 1KB',    fn () => json_decode('{"a":1,"b":[1,2,3,4,5]}', true));
```

```php
<?php
declare(strict_types=1);

// Instrumentação manual — útil para custom timings em produção
// (envia para StatsD/Prometheus/log estruturado)
final class Timer
{
    /** @var array<string, float> */
    private array $marks = [];
    /** @var array<string, float> */
    private array $durations = [];

    public function start(string $label): void
    {
        $this->marks[$label] = hrtime(true);
    }

    public function stop(string $label): float
    {
        $ns = hrtime(true) - ($this->marks[$label] ?? throw new \LogicException("no start: $label"));
        $this->durations[$label] = ($this->durations[$label] ?? 0) + $ns;
        return $ns / 1e6; // ms
    }

    public function report(LoggerInterface $log): void
    {
        $log->info('request.timings', array_map(
            static fn ($ns) => round($ns / 1e6, 2),
            $this->durations,
        ));
    }
}

// Uso em controller / middleware
$timer = new Timer();
$timer->start('db.users.find');
$user = $repo->find($id);
$timer->stop('db.users.find');

$timer->start('render.template');
$html = $twig->render('user.twig', ['user' => $user]);
$timer->stop('render.template');

$timer->report($logger);
```

```bash
# Workflow típico de investigação

# 1. APM (Datadog/New Relic) aponta endpoint suspeito:
#    /api/v1/dashboard p95 = 3.4s

# 2. Reproduzir em staging com dataset realista
curl -H "X-Profile: 1" -H "Cookie: SPX_ENABLED=1" https://staging.app.com/api/v1/dashboard

# 3. Abrir SPX UI, identificar hot function
#    Resultado: ReportBuilder::aggregate() = 71% do tempo

# 4. Profile localizado com Xdebug
XDEBUG_TRIGGER=1 php tests/repro_dashboard.php
qcachegrind /tmp/xdebug/cachegrind.out.*

# 5. Comparar antes/depois com Blackfire (CI)
blackfire run --reference=main vendor/bin/phpunit --filter DashboardTest
```

## Quando usar / quando não usar

- **Xdebug profile** — análise pontual em dev/staging, reproduzindo um caso específico. Nunca em produção.
- **SPX** — staging com tráfego real ou dev pesado; perfis recorrentes sem custo de licença.
- **Blackfire** — produção com baixo overhead, comparação entre PRs no CI, alertas em regressões de performance (>5% mais lento que baseline reprovam o PR).
- **Tideways** — APM contínuo com profile sempre-ligado em produção; alternativa ao New Relic para times PHP-only.
- **APM (Datadog/New Relic/Elastic)** — observabilidade transversal de toda a aplicação, *priorização* (qual endpoint profilear).
- **Não perfile** em dataset minúsculo — gargalos só aparecem com volume real. Use snapshot de produção (anonimizado) em staging.
- **Não perfile uma única request** para tirar conclusões definitivas. Métricas variam: GC, cache frio, contenção. Faça 30+ amostras.
- **Sampling profilers (php-spy, Datadog Continuous Profiler)** — sempre-ligados em produção, baixíssimo overhead, ideal para investigar problemas que só ocorrem sob carga real.

## Armadilhas comuns

1. **Otimizar por "intuição".** Trocar `foreach` por `array_map` "porque é mais rápido" — quase sempre é igual ou pior em PHP moderno. *Por quê*: micro-otimizações dependem do JIT, do tamanho do array, do tipo dos elementos. Sem medir, é fé.

2. **Xdebug ativo em produção (mesmo em `mode=off`).** A extensão carregada ainda adiciona overhead de hooks. *Por quê*: Xdebug intercepta funções internas via `zend_execute_ex` — mesmo desligado, há custo de verificação. Em prod, remova a extensão completamente.

3. **Confundir wall-clock com CPU time.** Uma função que faz `sleep(1)` aparece com 1s no profile, mas usa zero CPU. Em ambiente concorrente (workers), tempo de I/O pode ser sobreposto com outras requests. *Por quê*: profilers de instrumentação medem `microtime`, não `getrusage`.

4. **Profile com OPcache desativado.** Resultado mostra parsing/compile como hot — em produção (com OPcache) esses passos não existem. *Por quê*: pessoas desativam OPcache "para ver código atualizado" e esquecem de ativar antes de medir.

5. **Otimizar 2% do tempo total.** Função aparece como hot porque é chamada 10000 vezes — mas o total agregado é 30ms em request de 3s. ROI ridículo. *Por quê*: foco em "function que mais aparece" em vez de "categoria que mais consome".

6. **Dataset não-representativo.** Profile em conta com 5 produtos quando produção tem contas com 5000. A query que parecia O(1) é O(n²). *Por quê*: dados de dev raramente refletem distribuição real.

7. **Profile pós-cache.** Segunda request é toda servida por Redis, profile mostra "tudo rápido". O problema está na primeira request (cold cache). *Por quê*: medir só caminho otimizado esconde o caminho real.

8. **Não reaplicar profile pós-otimização.** Você corrige hot #1, mas hot #2 (antes invisível) agora domina. Otimização é iterativa: meça, corrija, meça de novo.

## Links relacionados

- [[OPcache-JIT-Preloading]]
- [[Fibers-N1]]
- [[Cache-PSR-6-16]]
- [[Filas]]
- [[FrankenPHP-RoadRunner-Swoole]]
- [[Observabilidade/Logs-Metricas-Tracing]]

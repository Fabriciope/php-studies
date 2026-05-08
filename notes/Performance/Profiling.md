---
tags: [php, performance, profiling]
fase: 6
status: stub
---

# Profiling: Xdebug, Blackfire, SPX

## Conceito

Três ferramentas, três casos de uso:

- **Xdebug** — debugger principal; também perfila (`xdebug.mode=profile`) gerando `.cachegrind` para QCachegrind. Lento, intrusivo. Para análise pontual.
- **Blackfire** — profiler comercial. Roda em produção com baixo overhead, gera flame graphs e comparações entre versões. Pago, mas excelente.
- **SPX** — profiler open-source (extensão), painel web embutido. Ótima alternativa gratuita ao Blackfire para staging/dev.

## Por que importa

Otimizar sem profile é loteria. 80% do tempo gasto em "performance" sem profile vai para o lugar errado. A regra: **sempre meça antes**, e meça em ambiente parecido com produção.

## Exemplo de código PHP

```ini
; xdebug — somente quando perfilar
xdebug.mode = profile
xdebug.output_dir = /tmp/xdebug
xdebug.start_with_request = trigger
; depois: ?XDEBUG_PROFILE=1 na request
```

```ini
; SPX — ativa via cookie/header
spx.http_enabled = 1
spx.http_key = "secret-spx-key"
spx.http_ip_whitelist = "127.0.0.1"
; UI em /?SPX_UI_URI=/
```

```php
<?php
// micro-bench manual — bom para validar otimização local
declare(strict_types=1);

function bench(string $label, callable $fn, int $iters = 100_000): void {
    $start = hrtime(true);
    for ($i = 0; $i < $iters; $i++) $fn();
    $ms = (hrtime(true) - $start) / 1e6;
    printf("%-30s %8.2f ms (%d iters)\n", $label, $ms, $iters);
}

bench('array_map fn', fn () => array_map(fn ($x) => $x * 2, range(1, 100)));
bench('foreach loop', function () {
    $out = [];
    foreach (range(1, 100) as $x) $out[] = $x * 2;
});
```

## Quando usar / quando não usar

- **Xdebug** — quando precisa step-debug ou perfil pontual; sempre **off** em produção.
- **SPX** — staging e dev, perfis frequentes, sem custo.
- **Blackfire** — produção com tráfego real; comparação de PRs no CI; profiles automáticos por endpoint.
- **Profiling em request única** raramente engana — mas hot-path de produção vê padrões diferentes (cache quente, dados maiores). Repita em prod-like.

## Armadilhas comuns

- Otimizar baseado em "intuição" (ex: trocar `foreach` por `array_map` "porque é mais rápido"). Quase sempre é igual ou pior. Meça.
- Esquecer Xdebug ativo em prod — penalty massiva.
- Confundir tempo total (wall-clock) com tempo CPU. I/O dorme, CPU não.
- Profile em dataset minúsculo: alvo errado. Use dataset realista.
- Otimizar 2% do tempo total — ROI ridículo. 80% do ganho está em 20% do código quente.

## Links relacionados

- [[OPcache-JIT-Preloading]]
- [[Fibers-N1]]
- [[Cache-PSR-6-16]]

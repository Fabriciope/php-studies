---
tags: [php, performance, opcache, jit]
fase: 6
status: stub
---

# OPcache, JIT, Preloading

## Conceito

- **OPcache** — cacheia o opcode compilado do PHP para que cada request não recompile arquivos. Em produção é **mandatório**: sem ele, performance cai 3–5x.
- **JIT** (8.0+) — compila opcodes "quentes" para código de máquina. Modos: `function` (mais conservador) e `tracing` (mais agressivo, geralmente melhor).
- **Preloading** (7.4+) — carrega arquivos uma vez no boot do FPM, residentes em memória compartilhada para todos os workers. Elimina autoload em cada request.

## Por que importa

A diferença entre app PHP "lenta" e "rápida" muitas vezes está aqui — não no código. App com OPcache desativado em produção é o erro de configuração mais frequente. JIT bem usado dá ganho real em código CPU-bound (parsing, math); em I/O-bound (típico web), ganho é menor mas não zero.

## Exemplo de código PHP

```ini
; php.ini para produção
opcache.enable = 1
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000
opcache.validate_timestamps = 0    ; deploy imutável; em dev = 1

opcache.preload = /app/preload.php
opcache.preload_user = www-data

; JIT
opcache.jit = tracing               ; ou 1255 / 1235
opcache.jit_buffer_size = 128M
```

```php
<?php
// preload.php — roda UMA VEZ no boot do FPM
declare(strict_types=1);

require __DIR__ . '/vendor/autoload.php';

// classes mais usadas (domínio + framework hot path)
$paths = [
    __DIR__ . '/src/Domain',
    __DIR__ . '/src/Application',
    __DIR__ . '/vendor/psr/http-message/src',
];

foreach ($paths as $dir) {
    /** @var SplFileInfo $file */
    foreach (new RecursiveIteratorIterator(new RecursiveDirectoryIterator($dir)) as $file) {
        if ($file->isFile() && $file->getExtension() === 'php') {
            opcache_compile_file($file->getRealPath());
        }
    }
}
```

## Quando usar / quando não usar

- **OPcache** — sempre, em qualquer ambiente além de "edição local".
- **`validate_timestamps=0`** apenas em produção com deploy imutável (container novo a cada release). Em dev = 1.
- **JIT tracing** — sempre, mas **meça**. Em apps muito I/O-bound o ganho é ≤5%.
- **Preloading** — quando o ganho de cold-start importa (FPM em ambiente com poucos workers, FaaS). Combine com [[FrankenPHP-RoadRunner-Swoole]] que já mantém estado.

## Armadilhas comuns

- `validate_timestamps=0` em dev → "alterei e não muda nada". Ative em dev.
- Preload de arquivo que tem `final readonly class X extends Y` onde Y não foi precarregado: erro fatal no boot.
- JIT buffer pequeno (`16M`) — JIT desliga silenciosamente. Use `128M+`.
- Esquecer de **reiniciar FPM** após mudar `php.ini` — config antiga continua valendo.

## Links relacionados

- [[Profiling]]
- [[FrankenPHP-RoadRunner-Swoole]]
- [[Fundamentos/Configuracao-PHP-INI]]

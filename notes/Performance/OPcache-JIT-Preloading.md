---
tags: [php, performance, opcache, jit]
fase: 6
status: draft
---

# OPcache, JIT, Preloading

## Conceito

O ciclo de execução de um script PHP atravessa quatro estágios bem definidos: **lexing** (tokenização do código-fonte em tokens via `token_get_all`), **parsing** (construção da AST), **compilação** (transformação da AST em opcodes — instruções de máquina virtual da Zend Engine) e **execução** (interpretação dos opcodes pelo Zend VM). Sem nenhum cache, cada request HTTP atendida pelo PHP-FPM repete os três primeiros estágios para *cada arquivo PHP envolvido na request*. Em uma aplicação Symfony ou Laravel realista, isso significa parsear e compilar centenas de arquivos a cada hit — desperdício gritante, já que o código mudou zero entre uma request e a próxima.

**OPcache** ataca exatamente esse desperdício. É uma extensão oficial (ativa por padrão desde PHP 5.5) que armazena os opcodes compilados em **memória compartilhada** (SHM, geralmente via `mmap` anônimo) acessível a todos os workers do FPM. Quando um arquivo é incluído (`require`, `include`, autoload), o OPcache verifica se já existe entrada em SHM; se sim, pula direto para a fase de execução. Sem OPcache, a perda de performance fica entre 3x e 5x — não é "otimização", é configuração básica de produção. A invalidação é por padrão baseada em `mtime` (`opcache.validate_timestamps=1`), mas em ambientes com deploy imutável (container novo a cada release) desativar essa validação economiza syscalls de `stat` por arquivo.

**Preloading**, introduzido no PHP 7.4 (RFC de Dmitry Stogov), vai além: ao iniciar o FPM, um script único (`opcache.preload`) é executado e todos os arquivos carregados ali ficam **permanentemente** em SHM, vinculados ao processo-pai e herdados por *fork* a todos os workers. Diferentemente do cache normal, classes precarregadas viram parte da definição "built-in" — não precisam ser autoloaded em nenhuma request. Há restrições rígidas: hierarquias de classes precisam ser totalmente resolvidas no preload (uma classe filha não pode ser precarregada se a pai não estiver), e mudanças no código preloaded exigem reload do FPM.

**JIT** (Just-In-Time, PHP 8.0+, também de Dmitry) compila opcodes "quentes" para **código de máquina nativo** (x86-64/ARM64 via DynASM), bypassando o Zend VM nas seções mais executadas. Há dois modos principais: **function JIT** (`opcache.jit=1205`, compila função inteira na primeira chamada) e **tracing JIT** (`opcache.jit=tracing` ou `1255`, monitora execução e compila apenas hot paths reais incluindo inlining cross-function). Tracing geralmente entrega ganhos maiores — em benchmarks CPU-bound como Mandelbrot ou parsers, ganho de 2-4x; em código web típico (I/O-bound), o ganho realista fica entre 3% e 10%, pois o tempo dominante está em syscalls, não em opcodes.

Historicamente, o ecossistema PHP olhou com inveja para o **HHVM** do Facebook (2011-2018), que abandonou PHP por Hack para alcançar JIT competitivo. Com OPcache+JIT+Preloading, o PHP oficial fechou boa parte dessa lacuna sem fragmentar a linguagem. Comparado a runtimes como V8 (JavaScript) ou HotSpot (Java), o JIT do PHP é mais simples — não há inlining especulativo nem deoptimization sofisticada — mas é suficiente para a maioria das cargas web.

## Por que importa

Os números falam por si. Em uma API Laravel típica respondendo `GET /users/{id}` (com cache de query):

| Configuração                                | RPS (1 worker) | p95   | CPU |
| ------------------------------------------- | -------------- | ----- | --- |
| Sem OPcache                                 | ~180           | 28ms  | 95% |
| OPcache padrão (`validate_timestamps=1`)    | ~620           | 9ms   | 78% |
| OPcache + `validate_timestamps=0`           | ~720           | 7ms   | 70% |
| OPcache + Preloading (200 classes hot)      | ~810           | 6ms   | 65% |
| OPcache + Preloading + JIT tracing          | ~860           | 5.5ms | 60% |

O salto sem OPcache → com OPcache é de 3.5x. O salto OPcache → +Preloading+JIT em web é de ~30%. **O erro mais caro em produção PHP é OPcache desligado** — geralmente por imagem Docker mal configurada ou `php.ini` herdado de dev. Em FaaS (AWS Lambda com `bref`) ou em ambientes com cold-start frequente, Preloading reduz o "primeiro hit" de 800ms para 200ms.

JIT brilha em código matemático, parsers, geradores de PDF, criptografia em PHP puro, simulações. Numa API REST que delega ao MySQL? Quase invisível — mas ainda é grátis (basta configurar) e não atrapalha.

## Exemplo de código PHP

```ini
; php.ini — configuração de PRODUÇÃO

[opcache]
opcache.enable                 = 1
opcache.enable_cli             = 0          ; CLI raramente compensa
opcache.memory_consumption     = 256        ; MB em SHM para opcodes
opcache.interned_strings_buffer = 32        ; MB para strings deduplicadas
opcache.max_accelerated_files  = 20000      ; > total de .php no projeto + vendor
opcache.max_wasted_percentage  = 10         ; reinicia se fragmentar > 10%
opcache.validate_timestamps    = 0          ; deploy imutável; em dev = 1
opcache.revalidate_freq        = 0          ; ignorado quando validate=0
opcache.save_comments          = 1          ; necessário para Doctrine/PHPUnit/atributos
opcache.fast_shutdown          = 1
opcache.huge_code_pages        = 1          ; mapeia em huge pages (THP), ganho ~5%

; Preloading
opcache.preload                = /app/preload.php
opcache.preload_user           = www-data

; JIT
opcache.jit                    = tracing    ; ou 1255
opcache.jit_buffer_size        = 128M       ; <64M = JIT silenciosamente desliga
opcache.jit_hot_loop           = 64         ; iterações antes de marcar loop como hot
opcache.jit_hot_func           = 127
```

```ini
; php.ini — configuração de DESENVOLVIMENTO

[opcache]
opcache.enable                 = 1
opcache.validate_timestamps    = 1
opcache.revalidate_freq        = 0          ; valida toda request (custa stat, mas dev)
opcache.preload                =            ; vazio: preload atrapalha hot reload
opcache.jit                    = off        ; JIT em dev confunde stack traces
```

```php
<?php
// preload.php — executado UMA VEZ no boot do FPM
declare(strict_types=1);

require __DIR__ . '/vendor/autoload.php';

// Estratégia 1: precarregar diretórios inteiros (classes do domínio)
$paths = [
    __DIR__ . '/src/Domain',
    __DIR__ . '/src/Application',
    __DIR__ . '/vendor/symfony/http-kernel/HttpKernel.php',
    __DIR__ . '/vendor/psr/http-message/src',
    __DIR__ . '/vendor/psr/log/src',
];

$compiled = 0;
$failed   = [];

foreach ($paths as $path) {
    if (is_file($path)) {
        opcache_compile_file($path) ? $compiled++ : $failed[] = $path;
        continue;
    }
    /** @var SplFileInfo $file */
    foreach (new RecursiveIteratorIterator(new RecursiveDirectoryIterator($path)) as $file) {
        if ($file->isFile() && $file->getExtension() === 'php') {
            try {
                opcache_compile_file($file->getRealPath()) ? $compiled++ : $failed[] = $file->getRealPath();
            } catch (\Throwable $e) {
                // classe com dependência não-resolvida; pular silenciosamente
                $failed[] = $file->getRealPath();
            }
        }
    }
}

error_log("preload: {$compiled} arquivos compilados, " . count($failed) . " falhas");
```

```php
<?php
// healthcheck.php — verifica saúde do OPcache em runtime
declare(strict_types=1);

$status = opcache_get_status(includeScripts: false);
$config = opcache_get_configuration();

$hitRate = $status['opcache_statistics']['opcache_hit_rate'];
$memUsed = $status['memory_usage']['used_memory'] / 1024 / 1024;
$memFree = $status['memory_usage']['free_memory'] / 1024 / 1024;
$wasted  = $status['memory_usage']['current_wasted_percentage'];

header('Content-Type: application/json');
echo json_encode([
    'hit_rate_pct'    => round($hitRate, 2),
    'memory_used_mb'  => round($memUsed, 1),
    'memory_free_mb'  => round($memFree, 1),
    'wasted_pct'      => round($wasted, 2),
    'cached_scripts'  => $status['opcache_statistics']['num_cached_scripts'],
    'jit_enabled'     => $status['jit']['enabled'] ?? false,
    'jit_buffer_used' => isset($status['jit']) ? $status['jit']['buffer_size'] - $status['jit']['buffer_free'] : 0,
], JSON_PRETTY_PRINT);
```

```bash
# Diagnóstico em produção
php -i | grep opcache.enable
php -r 'var_export(opcache_get_status(false)["opcache_statistics"]);'

# Reset manual (raríssimo; em geral, reload FPM)
php -r 'opcache_reset();'
systemctl reload php8.3-fpm
```

## Quando usar / quando não usar

- **OPcache: sempre.** Único cenário onde desativar faz sentido é instalação ultra-restrita de memória (<32MB total) — situação inexistente em servidores reais.
- **`opcache.validate_timestamps=0`** apenas com deploy imutável (Docker, atomic symlinks). Em servidores onde `git pull` altera código vivo, mantenha `1`.
- **Preloading** vale a pena quando: (a) o framework tem grafo de classes grande (Symfony, Laravel), (b) há tráfego suficiente para amortizar boot, (c) deploy é por reload completo do FPM. Em apps pequenas (<50 classes hot), ganho é estatisticamente irrelevante.
- **JIT tracing** sempre que possível, mas **meça**. Se a app é 99% I/O (microservice CRUD), o ganho de 3% pode não justificar a complexidade de debug.
- **JIT function (`1205`)** quando há código sensível a estabilidade (logs detalhados de stack, libs que inspecionam frames com `debug_backtrace`).
- **Não use** Preloading em ambiente serverless (Lambda) onde o handler é o próprio script — não há "boot único".

## Armadilhas comuns

1. **`validate_timestamps=0` em desenvolvimento.** Você altera código, recarrega a página, nada muda. Causa: OPcache nunca relê o arquivo. *Por quê acontece*: copia-se `php.ini` de produção para dev sem ajustar. Sempre force `validate_timestamps=1` em dev.

2. **`max_accelerated_files` baixo demais.** O default histórico é 10000; projetos modernos com vendor passam disso facilmente. Quando estoura, OPcache começa a **evictar** entradas (LRU) — hit rate cai, performance degrada silenciosamente. *Por quê*: dimensionamento sem inventário de `find . -name '*.php' | wc -l`.

3. **Preload de classe filha sem a pai.** `final class Foo extends Bar` precarregado antes de `Bar` → fatal no boot do FPM, todos workers morrem. *Por quê*: `opcache_compile_file` não resolve dependências, apenas compila. A ordem ou estratégia "compile_file em todos os arquivos depois `class_exists` para forçar resolução" importa.

4. **JIT buffer pequeno.** Configurar `jit_buffer_size=16M` em app grande → JIT desliga silenciosamente quando o buffer enche, sem warning. *Por quê*: economia falsa; 128MB de RSS extra é nada comparado ao perde-ganho de monitorar JIT.

5. **`save_comments=0` quebra Doctrine, atributos, PHPUnit.** A doc-comment é onde anotações vivem. Pré-PHP 8 era comum desligar para economizar memória; hoje é desastre. *Por quê*: tutoriais antigos ainda recomendam.

6. **Esquecer de recarregar FPM após mudar `php.ini`.** Configurações novas só aplicam após `systemctl reload php-fpm` (ou `kill -USR2`). *Por quê*: `php.ini` é lido apenas no boot do master FPM; workers herdam.

7. **OPcache em CLI sem necessidade.** Habilitar `opcache.enable_cli=1` para scripts curtos é gasto puro — o cache morre com o processo. Exceção: workers de fila long-running (use, vale a pena).

8. **`huge_code_pages=1` sem THP habilitado no kernel.** Setting silenciosamente ignorado. *Por quê*: depende de `/sys/kernel/mm/transparent_hugepage/enabled` estar como `always` ou `madvise`. Verifique antes de configurar.

## Links relacionados

- [[Profiling]]
- [[FrankenPHP-RoadRunner-Swoole]]
- [[Fundamentos/Configuracao-PHP-INI]]
- [[Cache-PSR-6-16]]
- [[Fibers-N1]]
- [[Infra/Docker-FPM]]

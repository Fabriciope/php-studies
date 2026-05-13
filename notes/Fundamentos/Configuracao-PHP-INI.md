---
tags: [php, fundamentos, runtime, infra]
fase: 1
status: draft
---

# Configuração php.ini

## Conceito

`php.ini` é o arquivo principal de configuração do runtime PHP. Controla limites de memória/tempo, comportamento de erro, sessões, OPcache, JIT, extensões carregadas, segurança de uploads, cookies. Cada SAPI (CLI, FPM, Apache mod_php) **pode ter seu próprio php.ini**: tipicamente `/etc/php/8.x/cli/php.ini` e `/etc/php/8.x/fpm/php.ini`. Conferir qual está em uso para qual contexto é a primeira lição de quem opera PHP em produção.

A distribuição oficial entrega dois templates: **`php.ini-development`** (verbose, display_errors=On, opcache validate_timestamps=1) e **`php.ini-production`** (silencioso, opcache agressivo, display_errors=Off, expose_php=Off). A diferença não é só "ligar/desligar coisas" — são duas posturas: dev é cooperativo com o programador, prod é defensivo com usuário e infraestrutura. Subir prod usando o template de dev é o erro mais comum em pipelines mal-feitos.

**Hierarquia de carregamento (ordem de precedência crescente)**:

1. `php.ini` principal (definido em compile-time, override por `-c` CLI ou `PHPRC` env).
2. Arquivos em `conf.d/` (ou `additional-ini`), ordem alfabética. Cada extensão tipicamente adiciona um (`20-opcache.ini`, `30-redis.ini`).
3. `.user.ini` (apenas em FPM/CGI, em diretório do script, refrescado conforme `user_ini.cache_ttl`).
4. `.htaccess` com `php_value`/`php_admin_value` (apenas Apache mod_php).
5. `ini_set()` em runtime (apenas diretivas `PHP_INI_USER` ou `PHP_INI_ALL`).

**Escopo de diretivas** (campo `changeable` no manual):
- `PHP_INI_SYSTEM` — só no `php.ini`. Exige restart do SAPI. Ex: `opcache.*`, `expose_php`.
- `PHP_INI_PERDIR` — `php.ini`, `.htaccess`, `.user.ini`. Ex: `upload_max_filesize`.
- `PHP_INI_USER` — qualquer um, incluindo `ini_set`. Ex: `memory_limit`, `error_reporting`.
- `PHP_INI_ALL` — qualquer lugar.

**Como inspecionar configuração efetiva**:

```bash
php --ini                # quais arquivos foram lidos
php -i | grep -i opcache # phpinfo() em texto, filtrável
php -r 'var_dump(ini_get("memory_limit"));'  # valor efetivo
```

```php
<?php
phpinfo();           // só em dev, NUNCA em prod exposto
ini_get_all('opcache'); // tudo de uma extensão
```

FPM tem ainda `pm.*` em arquivo separado (`pool.d/www.conf`) que é **pool** (não php.ini). Confundir os dois é frequente: `memory_limit` está em php.ini; `pm.max_children` em pool.conf.

## Por que importa

**Cenário 1 — vazamento de versão**: `expose_php=On` adiciona header `X-Powered-By: PHP/8.x.y`. Scanner automatizado identifica versão, casa com CVE conhecido, ataca. Desligar é grátis e elimina vetor.

**Cenário 2 — OPcache invalidando a cada request**: `opcache.validate_timestamps=1` checa `mtime` de cada arquivo PHP a cada hit (ou a cada `revalidate_freq` segundos). Em deploy imutável (container novo por release), isso é overhead puro. Trocar para `0` corta latência de cold start de bytecode.

**Cenário 3 — `display_errors=On` em prod**: stack trace na resposta HTTP vaza paths internos, nomes de classe, queries SQL parciais. Em conjunto com erros encadeados, vaza estrutura do banco. Atacante mapeia o sistema sem precisar de RCE.

**Cenário 4 — `memory_limit` mascarando vazamento**: subir de 256M para 1G "porque deu erro" trata sintoma. O sintoma volta com mais carga. Investigue com `memory_get_peak_usage()` e profilers (Blackfire, Xdebug) antes de aumentar.

**Cenário 5 — sessão sem `samesite`/`secure`**: cookie de sessão atravessa CSRF e roubo em rede insegura. Defaults históricos do PHP foram permissivos; production-template corrigiu, mas instalações antigas ainda têm vácuo.

## Exemplo de código PHP

**Template de produção comentado:**

```ini
;------------------------------------------------------------------
; SEGURANÇA
;------------------------------------------------------------------
expose_php = Off                 ; remove X-Powered-By
display_errors = Off             ; nunca em prod
display_startup_errors = Off     ; erros de extensão no boot também
log_errors = On
error_log = /var/log/php/error.log

allow_url_fopen = On             ; ok se controlado; Off se possível
allow_url_include = Off          ; SEMPRE Off — RCE clássico
disable_functions = exec,shell_exec,system,passthru,proc_open,popen

;------------------------------------------------------------------
; ERROS
;------------------------------------------------------------------
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT
log_errors_max_len = 4096
ignore_repeated_errors = On      ; reduz log spam

;------------------------------------------------------------------
; LIMITES
;------------------------------------------------------------------
memory_limit = 256M              ; ponto de partida; ajustar por workload
max_execution_time = 30          ; CLI ignora; FPM/Apache aplicam
max_input_time = 60
post_max_size = 8M
upload_max_filesize = 8M
max_input_vars = 1000            ; protege contra DoS por hash collision

;------------------------------------------------------------------
; OPCACHE — performance
;------------------------------------------------------------------
opcache.enable = 1
opcache.enable_cli = 0           ; só ligar se CLI longa (worker)
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000
opcache.validate_timestamps = 0  ; deploy imutável
opcache.revalidate_freq = 0
opcache.save_comments = 1        ; doctrine/symfony precisam
opcache.fast_shutdown = 1
opcache.preload = /app/preload.php
opcache.preload_user = www-data

;------------------------------------------------------------------
; JIT (PHP 8+)
;------------------------------------------------------------------
opcache.jit = tracing            ; tracing|function|disable
opcache.jit_buffer_size = 128M

;------------------------------------------------------------------
; REALPATH CACHE — evita stat() repetido
;------------------------------------------------------------------
realpath_cache_size = 4096K
realpath_cache_ttl = 600

;------------------------------------------------------------------
; SESSÕES
;------------------------------------------------------------------
session.cookie_secure = 1        ; só HTTPS
session.cookie_httponly = 1      ; não acessível por JS
session.cookie_samesite = "Lax"  ; ou "Strict"
session.use_strict_mode = 1      ; rejeita session IDs não emitidos
session.use_only_cookies = 1     ; não aceita ?PHPSESSID=
session.sid_length = 48
session.sid_bits_per_character = 6
session.gc_maxlifetime = 1440
session.gc_probability = 1
session.gc_divisor = 100

;------------------------------------------------------------------
; DATA/HORA
;------------------------------------------------------------------
date.timezone = "America/Sao_Paulo"
```

**Inspeção em runtime:**

```php
<?php
declare(strict_types=1);

// quais ini files o engine leu
var_dump(php_ini_loaded_file());     // string|false do php.ini principal
var_dump(php_ini_scanned_files());   // string com conf.d/*.ini ou false

// valor atual e tipo
var_dump(ini_get('memory_limit'));   // "256M"
var_dump(ini_get_all('opcache'));    // array completo com global/local

// override em runtime (apenas PHP_INI_USER/ALL)
ini_set('memory_limit', '512M');     // funciona
ini_set('opcache.enable', '0');      // FALHA SILENCIOSA — PHP_INI_SYSTEM
```

**Inspecionar escopo de uma diretiva:**

```bash
php -r 'print_r(ini_get_all("opcache")["opcache.enable"]);'
# Array (
#   [global_value] => 1
#   [local_value]  => 1
#   [access]       => 4     # 4 = PHP_INI_SYSTEM
# )
```

Valores de `access`: `1=USER`, `2=PERDIR`, `4=SYSTEM`, `7=ALL`.

**`.user.ini` em FPM:**

```ini
; /var/www/site/.user.ini
upload_max_filesize = 64M
post_max_size = 64M
```

Refrescado a cada `user_ini.cache_ttl` segundos (default 300). Não afeta CLI.

**Anti-padrão — `ini_set` em diretiva SYSTEM:**

```php
<?php
// inútil — opcache.* é PHP_INI_SYSTEM
ini_set('opcache.enable', '1');

// também inútil em runtime — extensão já está carregada ou não
ini_set('zend_extension', 'opcache');
```

Mudanças assim falham **sem erro**. Verifique com `ini_get()` se realmente mudou.

**Tabela de categorias e diretivas-chave:**

| Categoria       | Diretivas críticas                                                |
|-----------------|-------------------------------------------------------------------|
| Segurança       | `expose_php`, `display_errors`, `allow_url_include`, `disable_functions`, `open_basedir` |
| Performance     | `opcache.*`, `realpath_cache_size`, `realpath_cache_ttl`, `memory_limit` |
| Erros           | `error_reporting`, `log_errors`, `error_log`, `display_startup_errors` |
| Sessões         | `session.cookie_secure`, `samesite`, `use_strict_mode`, `gc_*`     |
| Uploads/Limites | `post_max_size`, `upload_max_filesize`, `max_input_vars`, `max_execution_time` |
| Localização     | `date.timezone`, `intl.default_locale`, `mbstring.internal_encoding` |

## Quando usar / quando não usar

- **Sempre** partir do `php.ini-production` em produção. Custa zero, ganha defesa em profundidade.
- **Sempre** ter `error_log` apontando para arquivo monitorado (logrotate + agente APM).
- **`opcache.validate_timestamps=0`** apenas se deploy é imutável (container novo, ou `opcache_reset()` após release). Em dev, mantenha `1` ou setá-lo curto via `revalidate_freq`.
- **`opcache.preload`** vale a pena em apps com hot path estável (Symfony, Laravel). Custa boot mais lento, ganha latência menor por request.
- **JIT (`opcache.jit=tracing`)** ganha em CPU-bound (cálculo, parsing). Em I/O-bound (típico web), ganho é marginal — mas raramente piora.
- **`max_execution_time`** não cobre `sleep()` nem chamadas de sistema bloqueantes em alguns SAPIs. Complemente com timeout no client HTTP/DB.
- **Não aumentar `memory_limit`** sem investigar a causa. Use `memory_get_peak_usage()`, profilers.
- **Não desabilitar `opcache`** em CLI worker de longa duração — perde a aceleração. Ligue `opcache.enable_cli=1`.
- **Não confiar em `disable_functions`** como sandbox. É defense-in-depth, não containment.

## Armadilhas comuns

- **Dois `php.ini` diferentes**: CLI e FPM têm arquivos separados; mudar um não afeta o outro. Causa: cada SAPI carrega seu próprio. Verifique com `php --ini` e `phpinfo()` via web.
- **`display_errors=Off` mas `display_startup_errors=On`** ainda vaza no boot da extensão. Causa: são diretivas independentes; o template antigo deixava startup ligado.
- **`phpinfo()` em prod**: arquivo `info.php` esquecido na raiz mostra config inteira, paths, extensões, valores de env. Vazamento crítico. Causa: hábito de dev nunca removido.
- **Mudança ignorada por escopo errado**: `ini_set('opcache.enable', 0)` falha silenciosamente porque é `PHP_INI_SYSTEM`. Causa: o engine não lança erro em set fora de escopo; só ignora.
- **FPM não reiniciou**: editou `php.ini`, mas não rodou `systemctl reload php-fpm` (ou `kill -USR2 <master>`). Causa: FPM carrega ini só no startup; reload re-lê sem derrubar conexões.
- **`opcache.memory_consumption` baixo**: app grande satura cache, gera evictions, vira "cache miss" perpétuo. Monitorar `opcache_get_status()['memory_usage']`. Causa: default de 128M não cobre apps reais.
- **`opcache.save_comments=0` quebra anotações**: Doctrine, Symfony, frameworks que dependem de PHPDoc/annotations perdem metadados. Causa: anotações em comentários não chegam ao reflection.
- **`realpath_cache_size` esquecido em apps com muitos `require`**: cada `include_path` vira `stat()` repetido. Subir para 4M+ tipicamente reduz latência em apps com vendor pesado. Causa: cache default (16K) é pequeno demais para Composer trees.
- **`post_max_size` < `upload_max_filesize`**: upload maior que `post_max_size` falha **antes** de `upload_max_filesize` ser checado, e `$_POST` vem vazio sem erro óbvio. Causa: post_max é envelope; upload_max é conteúdo. Sempre post_max >= upload_max.
- **`max_input_vars=1000` engole arrays grandes em form**: form com checkbox de muitos itens silenciosamente perde dados além do 1000º. Causa: proteção contra DoS por hash collision (CVE-2011-4885); ajustar conscientemente.
- **`session.gc_probability=0`** em produção: GC nunca roda, sessões expiram só por timeout do filesystem. Em alguns distros, cron externa o GC; em outras, fica para o PHP. Causa: defaults variam por pacote.
- **`open_basedir` quebra `tempnam` e libs**: confina paths, mas frameworks que escrevem em `/tmp` ou cache fora da raiz falham. Causa: restrição se aplica antes da abertura, sem fallback.
- **`date.timezone` ausente**: cada `date()` emite warning ou usa UTC. Causa: PHP 5.x mudou o default; quem não setou herda comportamento inconsistente.
- **`.user.ini` cacheado**: mudanças não aparecem por 5min default (`user_ini.cache_ttl`). Em dev, baixe para 0. Causa: PHP evita ler o arquivo a cada request por performance.

## Links relacionados

- [[Performance/OPcache-JIT-Preloading]]
- [[Infra/Docker-FPM]]
- [[Infra/Observabilidade-OWASP]]
- [[Hierarquia-Throwable]]
- [[Strict-Types]]
- [[Padroes/SOLID]]

---
tags: [php, fundamentos, runtime, infra]
fase: 1
status: stub
---

# Configuração php.ini

## Conceito

`php.ini` controla o runtime: limites de memória/tempo, OPcache, JIT, error handling, sessions. Em produção, configurar errado custa em performance, segurança ou estabilidade. Há diretivas com escopo `PHP_INI_SYSTEM` (só ini), `PHP_INI_PERDIR` (.htaccess/`fpm.conf`) e `PHP_INI_USER` (em runtime via `ini_set`).

## Por que importa

Default do PHP é "dev-friendly". Em produção `display_errors=On` vaza stack trace, `opcache.validate_timestamps=1` mata performance, `expose_php=On` revela versão. Auditar `php.ini` é higiene básica.

## Exemplo de código PHP

```ini
; produção — essencial

; segurança
expose_php = Off
display_errors = Off
log_errors = On
error_log = /var/log/php/error.log

; limites
memory_limit = 256M
max_execution_time = 30
post_max_size = 8M
upload_max_filesize = 8M

; opcache
opcache.enable = 1
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000
opcache.validate_timestamps = 0      ; em prod com deploy imutável
opcache.preload = /app/preload.php
opcache.preload_user = www-data

; jit
opcache.jit = tracing
opcache.jit_buffer_size = 128M

; sessões
session.cookie_secure = 1
session.cookie_httponly = 1
session.cookie_samesite = "Lax"
session.use_strict_mode = 1
```

## Quando usar / quando não usar

- **`opcache.validate_timestamps=0`** apenas se o deploy é imutável (container novo a cada release). Em dev, mantenha `=1`.
- **`memory_limit`** alto (>512M) costuma esconder vazamento — investigue antes de aumentar.
- **`max_execution_time`** não cobre operações I/O bloqueantes em alguns SAPIs; complemente com timeout no client HTTP.

## Armadilhas comuns

- `display_errors=Off` mas `display_startup_errors=On` ainda vaza no boot.
- `phpinfo()` deixado em produção — clássico vazamento de configuração.
- Mudanças em ini com `ini_set` que **não funcionam** porque a diretiva é `PHP_INI_SYSTEM` (ex.: `opcache.*`).
- Esquecer de reiniciar FPM após editar `php.ini` (`systemctl reload php-fpm` ou `kill -USR2`).

## Links relacionados

- [[Performance/OPcache-JIT-Preloading]]
- [[Infra/Docker-FPM]]
- [[Infra/Observabilidade-OWASP]]

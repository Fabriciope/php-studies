---
tags: [php, fundamentos, fase_1]
fase: 1
status: stub
---

# php.ini

## Conceito

`php.ini` configura o comportamento do PHP: exibição de erros, timezone, memória, upload, extensões e limites de execução.

## Por que importa

Entender configuração evita investigar bugs que são, na verdade, diferença de ambiente entre CLI, servidor local e produção.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

echo 'memory_limit=' . ini_get('memory_limit') . PHP_EOL;
echo 'timezone=' . date_default_timezone_get() . PHP_EOL;
```

## Quando usar / quando não usar

Consulte durante setup e diagnóstico. Não dependa de configuração local implícita para regras do programa.

## Armadilhas comuns

- Alterar `php.ini` errado: CLI e servidor podem usar arquivos diferentes.
- Desenvolver com erros ocultos.
- Não definir timezone em aplicações que lidam com datas.

## Links relacionados

[[Sintaxe básica]] [[Datas]] [[Erros e exceções]]

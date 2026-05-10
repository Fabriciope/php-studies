---
tags: [php, composer, fase_6]
fase: 6
status: stub
---

# Autoload PSR-4

## Conceito

PSR-4 é uma convenção de autoload que relaciona prefixo de namespace com diretório. Exemplo: `App\` aponta para `src/`.

## Por que importa

Ela permite usar classes sem `require` manual e ensina a mapear nome de classe, namespace e caminho físico.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

namespace App;

final class Calculator
{
    public static function double(int $value): int
    {
        return $value * 2;
    }
}
```

## Quando usar / quando não usar

Use quando classes estiverem em arquivos separados. Não use autoload para esconder uma estrutura confusa; namespace deve refletir organização simples.

## Armadilhas comuns

- Arquivo em `src/Tools/Calculator.php` com namespace apenas `App`.
- Classe com nome diferente do arquivo em sistemas sensíveis a maiúsculas/minúsculas.
- Editar `composer.json` e não regenerar autoload.

## Links relacionados

[[composer.json]] [[O que é Composer]] [[Classes e objetos]] [[Funções]]

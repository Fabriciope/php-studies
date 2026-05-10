---
tags: [php, composer, fase_6]
fase: 6
status: stub
---

# O que é Composer

## Conceito

Composer é o gerenciador de dependências e autoload mais usado no ecossistema PHP. Para fundamentos, ele deve ser entendido primeiro como ferramenta de organização de arquivos e classes.

## Por que importa

Ele evita `require` manual espalhado, padroniza scripts do projeto e prepara o código para bibliotecas externas sem abandonar o entendimento da linguagem.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

require __DIR__ . '/vendor/autoload.php';

use App\Calculator;

echo Calculator::double(21) . PHP_EOL;
```

## Quando usar / quando não usar

Use quando o exercício já tiver múltiplos arquivos, namespaces ou dependências. Não use pacote externo para resolver algo que você ainda está tentando aprender na linguagem base.

## Armadilhas comuns

- Rodar código com `vendor/autoload.php` ausente.
- Instalar bibliotecas antes de entender a operação básica em PHP puro.
- Confundir Composer com framework.

## Links relacionados

[[composer.json]] [[composer.lock]] [[Autoload PSR-4]] [[Scripts Composer]] [[Funções]]

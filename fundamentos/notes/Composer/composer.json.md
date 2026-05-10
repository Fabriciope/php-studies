---
tags: [php, composer, fase_6]
fase: 6
status: stub
---

# composer.json

## Conceito

`composer.json` descreve metadados, dependências, autoload e scripts de um projeto PHP.

## Por que importa

Ele torna o projeto reproduzível e ensina organização mínima: onde ficam as classes, quais comandos existem e quais pacotes são necessários.

## Exemplo de código PHP

```php
{
  "autoload": {
    "psr-4": { "App\\": "src/" }
  },
  "scripts": {
    "start": "php public/index.php"
  }
}
```

## Quando usar / quando não usar

Use assim que houver mais de um arquivo ou classe. Não complique com dezenas de scripts no início; prefira poucos comandos que você realmente executa.

## Armadilhas comuns

- Esquecer de rodar `composer dump-autoload` depois de mudar autoload.
- Escrever namespace diferente do caminho configurado.
- Colocar dependência de desenvolvimento em `require`.

## Links relacionados

[[O que é Composer]] [[composer.lock]] [[Autoload PSR-4]] [[require vs require-dev]] [[Scripts Composer]]

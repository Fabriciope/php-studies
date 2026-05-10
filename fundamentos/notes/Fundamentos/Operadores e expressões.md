---
tags: [php, fundamentos, fase_2]
fase: 2
status: stub
---

# Operadores e expressões

## Conceito

Expressões produzem valores; operadores combinam valores por aritmética, comparação, lógica, concatenação, coalescência ou atribuição.

## Por que importa

Dominar expressões evita condicionais ambíguas e bugs de comparação, especialmente quando dados chegam como string.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

$price = 1000;
$discount = 150;
$hasStock = true;

$canBuy = $hasStock && ($price - $discount) > 0;
var_dump($canBuy);
```

## Quando usar / quando não usar

Use `===`, `!==`, `??` e parênteses para deixar intenção clara. Evite encadear expressões longas que escondem regras importantes.

## Armadilhas comuns

- Usar `==` e aceitar coerção inesperada.
- Misturar concatenação e soma sem parênteses.
- Usar ternários aninhados quando `match` ou função nomeada seria mais claro.

## Links relacionados

[[Variáveis e tipos]] [[Controle de fluxo]] [[Sistema de tipos]]

---
tags: [php, fundamentos, fase_2]
fase: 2
status: stub
---

# Variáveis e tipos

## Conceito

Variáveis guardam valores em tempo de execução; tipos descrevem o formato desses valores, como `int`, `float`, `string`, `bool`, `array`, `object`, `null` e recursos internos.

## Por que importa

Grande parte dos bugs em PHP nasce de suposições erradas sobre o tipo real de um valor recebido de formulário, JSON, CLI ou arquivo.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

$rawQuantity = '10';
$quantity = (int) $rawQuantity;
$totalCents = $quantity * 1299;

var_dump($rawQuantity, $quantity, $totalCents);
```

## Quando usar / quando não usar

Use tipos explícitos em parâmetros e retornos. Não confie em coerção automática para dado vindo de fora da aplicação.

## Armadilhas comuns

- Achar que `'10'` e `10` são sempre equivalentes.
- Usar `empty()` sem entender que `'0'` é considerado vazio.
- Aceitar `mixed` sem normalizar o valor logo na borda do programa.

## Links relacionados

[[Sistema de tipos]] [[strict_types]] [[Operadores e expressões]] [[Funções]]

---
tags: [php, fundamentos, fase_4]
fase: 4
status: stub
---

# Arrays

## Conceito

Arrays em PHP servem como listas ordenadas e mapas associativos. Essa flexibilidade é poderosa, mas exige disciplina sobre formato esperado.

## Por que importa

Muitos programas PHP manipulam JSON, formulários e resultados de banco como arrays; saber validar chaves e tipos evita warnings e dados corrompidos.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

$items = [
    ['sku' => 'A1', 'quantity' => 2, 'price' => 1500],
    ['sku' => 'B2', 'quantity' => 1, 'price' => 990],
];

$total = array_sum(array_map(
    fn (array $item): int => $item['quantity'] * $item['price'],
    $items,
));
```

## Quando usar / quando não usar

Use arrays para dados simples e transitórios. Quando o mesmo array exige validação repetida de chaves, considere uma função normalizadora ou objeto pequeno.

## Armadilhas comuns

- Acessar chave inexistente sem `array_key_exists()` ou validação.
- Misturar lista e mapa no mesmo array.
- Passar arrays grandes entre funções sem documentar formato.

## Links relacionados

[[Strings]] [[Funções]] [[Sistema de tipos]] [[Strings]]

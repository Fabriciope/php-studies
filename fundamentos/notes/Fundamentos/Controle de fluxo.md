---
tags: [php, fundamentos, fase_3]
fase: 3
status: stub
---

# Controle de fluxo

## Conceito

Controle de fluxo decide quais instruções executam: `if`, `else`, `match`, `for`, `foreach`, `while`, `break`, `continue` e retorno antecipado.

## Por que importa

Fluxo claro transforma regras de negócio em caminhos previsíveis e reduz estados impossíveis.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

$status = 'paid';

$message = match ($status) {
    'paid' => 'Pedido pago',
    'open' => 'Pedido em aberto',
    'cancelled' => 'Pedido cancelado',
    default => throw new InvalidArgumentException('Status inválido'),
};
```

## Quando usar / quando não usar

Use `match` para mapeamentos exatos e `if` para condições compostas. Não use `switch` por hábito quando comparação estrita for importante.

## Armadilhas comuns

- Esquecer `default` em valores vindos de fora.
- Colocar regra complexa diretamente dentro de `if`.
- Usar loop quando uma função de array deixaria a transformação mais clara.

## Links relacionados

[[Operadores e expressões]] [[Funções]] [[Erros e exceções]]

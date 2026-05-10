---
tags: [php, fundamentos, fase_2]
fase: 2
status: stub
---

# strict_types

## Conceito

`declare(strict_types=1)` altera a verificação de tipos escalares no arquivo chamador, reduzindo coerções automáticas em chamadas de função.

## Por que importa

Ele força disciplina: se a função pede `int`, passar string numérica por acidente deixa de ser comportamento silencioso.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

function double(int $value): int
{
    return $value * 2;
}

// double('21'); // TypeError com strict_types=1 no arquivo chamador.
```

## Quando usar / quando não usar

Use em todos os arquivos de estudo e produção novos. Não use como substituto para validar entrada externa; JSON, CLI e formulários ainda chegam sem confiança.

## Armadilhas comuns

- Achar que `strict_types` vale globalmente para o projeto inteiro.
- Esperar que ele valide tipos internos de arrays.
- Remover validação de borda porque a função tem tipo declarado.

## Links relacionados

[[Sistema de tipos]] [[Funções]] [[Throwable]]

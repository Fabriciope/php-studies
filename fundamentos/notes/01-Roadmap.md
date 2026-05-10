---
tags: [php, roadmap, fundamentos]
fase: 0
status: stub
---

# 01-Roadmap

## Conceito

Roadmap de 16 semanas focado em consolidar PHP básico e moderno antes de avançar para abstrações maiores.

## Por que importa

A progressão evita o erro comum de estudar padrões, APIs e infraestrutura sem entender coerção de tipos, arrays, escopo, exceções e autoload.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

$fases = [
    1 => 'Ambiente e sintaxe',
    2 => 'Tipos e strict_types',
    3 => 'Fluxo e funções',
    4 => 'Arrays e strings',
];
```

## Quando usar / quando não usar

Use para planejar entregáveis quinzenais. Não use como calendário rígido se tipos, funções ou arrays ainda estiverem frágeis.

## Armadilhas comuns

- Avançar sem conseguir explicar o tipo de cada variável.
- Usar classes para esconder funções mal compreendidas.
- Adiar Composer até o projeto ficar desorganizado.

## Links relacionados

[[00-Index]] [[Sistema de tipos]] [[strict_types]] [[Funções]] [[Arrays]] [[composer.json]]

---
tags: [php, fundamentos, fase_3]
fase: 3
status: stub
---

# Funções

## Conceito

Funções nomeiam uma operação, recebem parâmetros, podem declarar tipos, retornam valores e criam uma fronteira pequena para raciocinar sobre comportamento.

## Por que importa

Funções bem tipadas são o principal instrumento para consolidar PHP básico: elas obrigam você a definir entrada, saída, erro e responsabilidade.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

function subtotalInCents(int $unitPrice, int $quantity): int
{
    if ($quantity <= 0) {
        throw new InvalidArgumentException('Quantidade deve ser positiva.');
    }

    return $unitPrice * $quantity;
}
```

## Quando usar / quando não usar

Use funções para regras pequenas e testáveis. Não use função genérica com muitos parâmetros opcionais quando duas funções específicas comunicariam melhor a intenção.

## Armadilhas comuns

- Omitir tipo de retorno.
- Retornar `false`, `null`, string ou array para representar erros diferentes.
- Alterar variável global dentro da função sem necessidade.

## Links relacionados

[[strict_types]] [[Sistema de tipos]] [[Erros e exceções]] [[First-class callable]]

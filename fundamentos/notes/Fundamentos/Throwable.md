---
tags: [php, fundamentos, fase_5]
fase: 5
status: stub
---

# Throwable

## Conceito

`Throwable` é a interface implementada por `Exception` e `Error`, permitindo capturar falhas lançáveis em PHP.

## Por que importa

Ela ajuda a entender a hierarquia de falhas, mas deve ser usada com cuidado para não esconder erros de programação.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

try {
    $result = subtotalInCents(1000, 0);
} catch (Throwable $error) {
    fwrite(STDERR, $error->getMessage() . PHP_EOL);
}
```

## Quando usar / quando não usar

Use `Throwable` em fronteiras do programa, como comando CLI ou ponto de entrada HTTP. Dentro do domínio, prefira capturar exceções específicas.

## Armadilhas comuns

- Capturar `Throwable` em toda função.
- Continuar execução após erro crítico sem estado confiável.
- Exibir stack trace para usuário final.

## Links relacionados

[[Erros e exceções]] [[Funções]] [[strict_types]]

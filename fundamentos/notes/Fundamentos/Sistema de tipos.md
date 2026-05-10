---
tags: [php, fundamentos, fase_2]
fase: 2
status: stub
---

# Sistema de tipos

## Conceito

O sistema de tipos permite declarar parâmetros, retornos, propriedades e combinações como union types, nullable types e `mixed`.

## Por que importa

Tipos são documentação executável: reduzem ambiguidades e ajudam ferramentas como PHPStan a encontrar bugs antes do runtime.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

function findEmail(array $payload): string|null
{
    $email = $payload['email'] ?? null;

    return is_string($email) && filter_var($email, FILTER_VALIDATE_EMAIL)
        ? $email
        : null;
}
```

## Quando usar / quando não usar

Use tipos para tornar contratos explícitos. Não use `mixed` como saída permanente; normalize para tipo específico o mais cedo possível.

## Armadilhas comuns

- Declarar `array` sem saber o formato interno.
- Usar `?string` quando ausência e string vazia precisam ser diferenciadas.
- Ignorar `TypeError` como sinal de contrato violado.

## Links relacionados

[[Variáveis e tipos]] [[strict_types]] [[Arrays]] [[Funções]]

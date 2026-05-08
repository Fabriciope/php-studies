---
tags: [php, fundamentos, tipos]
fase: 1
status: stub
---

# Strict Types

## Conceito

`declare(strict_types=1)` muda o modo de coerção de argumentos e retornos: em vez de o PHP converter silenciosamente `"42"` em `42`, ele lança `TypeError`. A diretiva é **por arquivo** e afeta apenas chamadas feitas **a partir** desse arquivo, não chamadas recebidas.

## Por que importa

Sem strict, a função `add(int $a, int $b)` aceita `add("3", "4.9")` e devolve `7`. Aceitar entrada errada é a maneira mais barata de empurrar bug pra frente. Strict transforma o sistema de tipos de "sugestão" em "contrato".

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

function add(int $a, int $b): int {
    return $a + $b;
}

add(1, 2);       // 3
add("1", "2");   // TypeError — esperado int, recebido string
```

Sem `strict_types`, a segunda chamada retornaria `3` silenciosamente.

## Quando usar / quando não usar

- **Sempre usar** em código novo. Primeira linha de todo `.php` do projeto.
- **Atenção legacy:** ativar em arquivo que recebia `"1"` de um form e tratava como int vai quebrar — mas esse "quebrar" é a feature.
- **Nunca pular** em domínio. Especialmente em Value Objects.

## Armadilhas comuns

- A diretiva é **por arquivo de chamada**, não da função chamada. Se `A.php` tem strict e `B.php` não, ao chamar de `B.php` para `A.php`, vale o modo de `B.php`.
- `int` aceita `float` sem casas? **Não** em strict — `add(1.0, 2)` quebra.
- `null` continua precisando de `?int` ou union `int|null`.

## Links relacionados

- [[Sistema-de-Tipos]]
- [[Hierarquia-Throwable]]
- [[OOP-Moderno/Readonly-Classes]]

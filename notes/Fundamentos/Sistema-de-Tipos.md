---
tags: [php, fundamentos, tipos]
fase: 1
status: stub
---

# Sistema de Tipos

## Conceito

PHP 8.x tem um sistema de tipos rico — escalares, união (`A|B`), interseção (`A&B`), `never`, `void`, `mixed`, `self`, `static`, `iterable`, `true`/`false` literais, `null` em union. Combinados com PHPStan/Psalm, dão suporte ao mesmo nível de design-by-contract de linguagens estáticas.

## Por que importa

Tipos não são para o runtime — são para o **leitor humano** e para a análise estática. Um `Result<Order, NotFoundError>` (via union `Order|NotFoundError`) elimina classes inteiras de bug que `try/catch` ainda permite.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

interface Logger { public function log(string $msg): void; }
interface Tagger { public function tag(string $key): void; }

// intersection: o argumento precisa implementar AMBOS
function notify(Logger&Tagger $sink, string $msg): void {
    $sink->tag('notify');
    $sink->log($msg);
}

// união como retorno: sucesso OU falha tipada
function findOrder(int $id): Order|NotFoundError {
    return $id > 0 ? new Order($id) : new NotFoundError("id={$id}");
}

// never: função que nunca retorna (loop infinito ou throw)
function fail(string $msg): never {
    throw new RuntimeException($msg);
}
```

## Quando usar / quando não usar

- **União (`A|B`)** para resultados sucesso/falha tipados — alternativa a exceções de fluxo.
- **Interseção (`A&B`)** quando o consumidor precisa de duas capacidades; evita `instanceof` no meio.
- **`never`** para funções que sempre lançam; ajuda análise de fluxo.
- **`mixed`** evite — é "desisti de tipar". Use `string|int|array` se for de fato 3 coisas.

## Armadilhas comuns

- Confundir `void` (sem retorno) com `null` (retorna null).
- `self` é o tipo da classe **declarada**, `static` é o **chamador efetivo**. Em fluent interfaces use `static`.
- Union com classes desabilita `match` exaustivo — por isso prefira Enums quando possível.
- `iterable` aceita array E Generator; quando precisa SÓ de array, use `array`.

## Links relacionados

- [[Strict-Types]]
- [[OOP-Moderno/Enums]]
- [[Padroes/SOLID]]

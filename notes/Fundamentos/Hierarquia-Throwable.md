---
tags: [php, fundamentos, excecoes]
fase: 1
status: stub
---

# Hierarquia Throwable

## Conceito

Em PHP, `Throwable` é a interface raiz de tudo que pode ser `throw`-ado. Abaixo dela há dois ramos:

- **`Error`** — falhas de runtime do motor (`TypeError`, `ParseError`, `DivisionByZeroError`). Não são para você capturar em fluxo normal.
- **`Exception`** — falhas de domínio/aplicação que você define e captura.

## Por que importa

Misturar os dois é o jeito mais rápido de transformar bugs em comportamento "tratado". Capturar `Throwable` no controller engole `TypeError` e o sistema continua mentindo. A regra: **`Error` é bug, `Exception` é caso de negócio**.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

abstract class DomainException extends \RuntimeException {}

final class OrderNotFound extends DomainException {
    public static function withId(int $id): self {
        return new self("order {$id} not found");
    }
}

final class InvalidStateTransition extends DomainException {}

// no use case:
try {
    $order = $repo->find($id) ?? throw OrderNotFound::withId($id);
    $order->pay($amount);
} catch (DomainException $e) {
    // 4xx → resposta de negócio
    return Response::badRequest($e->getMessage());
}
// Errors propagam → 500 → alerta no Sentry
```

## Quando usar / quando não usar

- **Hierarquia própria de domínio** (`DomainException` como base) — sempre.
- **`catch (\Throwable)`** apenas no boundary externo (entrypoint HTTP, worker), e mesmo assim para **logar e relançar/converter**.
- **Não usar exceções como fluxo de controle** (ex.: "se não achar, throw"). Use union `Order|null` ou `Order|NotFoundError`.

## Armadilhas comuns

- `try/catch (\Exception)` **não pega** `TypeError` (é `Error`). Esse silêncio às vezes é desejado.
- `finally` sempre executa, inclusive após `return` — útil para cleanup, perigoso se você `return` dentro dele.
- Não enriquecer a exception com contexto (`extends` sem propriedades) — perde-se rastreio.

## Links relacionados

- [[Strict-Types]]
- [[Sistema-de-Tipos]]
- [[Infra/Observabilidade-OWASP]]

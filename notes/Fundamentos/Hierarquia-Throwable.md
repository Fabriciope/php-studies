---
tags: [php, fundamentos, excecoes]
fase: 1
status: draft
---

# Hierarquia Throwable

## Conceito

Em PHP, **toda** coisa lançável implementa a interface `Throwable`. Essa interface foi introduzida no PHP 7.0 para unificar os antigos `Exception` (camada de usuário) e os novos `Error` (falhas do engine, antes "fatais e incapturáveis"). A partir daí, `try { ... } catch (\Throwable $t)` captura literalmente tudo lançável — bug ou negócio.

Hierarquia canônica:

```
Throwable (interface)
├── Error
│   ├── TypeError              (8.0+ inclui ArgumentCountError)
│   │   └── ArgumentCountError
│   ├── ValueError             (PHP 8.0)
│   ├── ArithmeticError
│   │   └── DivisionByZeroError
│   ├── AssertionError
│   ├── ParseError
│   ├── CompileError
│   │   └── ParseError         (PHP 7.3+ herda de CompileError)
│   └── UnhandledMatchError    (PHP 8.0)
└── Exception
    ├── ErrorException
    ├── LogicException
    │   ├── BadFunctionCallException
    │   │   └── BadMethodCallException
    │   ├── DomainException
    │   ├── InvalidArgumentException
    │   ├── LengthException
    │   └── OutOfRangeException
    └── RuntimeException
        ├── OutOfBoundsException
        ├── OverflowException
        ├── RangeException
        ├── UnderflowException
        └── UnexpectedValueException
```

**Modelo mental**: o lado `Error` é "o engine te chamou a atenção" (você passou tipo errado, dividiu por zero, escreveu sintaxe inválida, retornou tipo errado). O lado `Exception` é "minha lógica encontrou estado inesperado". A distinção é semântica, não técnica — captura funciona em ambos. Mas catch-all em `Exception` **não captura `Error`**; esse é o ponto.

**Subdivisão de `Exception` da SPL**: a Standard PHP Library propõe a dicotomia `LogicException` × `RuntimeException`. **Logic** = bug do programador (pré-condição violada, chamou método em estado inválido). **Runtime** = condição externa imprevisível (rede caiu, arquivo sumiu, banco rejeitou query). A diferença é se a causa está no código ou no mundo.

**Encadeamento (chaining)**: cada Throwable tem um `previous` opcional (`Throwable::getPrevious()`), passado no construtor. Permite rewrap: captura uma `PDOException` no repositório, lança uma `UserRepositoryFailure` carregando a original como contexto. Cadeia preservada, contexto adicionado.

**`throw` como expressão (PHP 8.0)**: antes, `throw` era statement; precisava de `if` para usar. Agora é expressão e cabe em `??`, ternário, arrow function. Mudança pequena em ergonomia, grande em legibilidade.

**`finally`**: sempre executa, mesmo após `return`, mesmo após exception não capturada. Útil para fechar recurso. Perigoso se você `return` dentro do `finally` — sobrescreve o `return` do `try` e silencia exceptions pendentes.

## Por que importa

**Cenário 1 — catch genérico engole bugs**: equipe coloca `catch (\Throwable $t) { logger->error($t); }` no controller. `TypeError` por argumento errado vira log silencioso; produção continua respondendo 200 OK com dados parciais. Distinguir `Error` (sobe, vira 500, dispara alerta) de `Exception` (4xx esperado) é vital.

**Cenário 2 — exception como controle de fluxo**: `function find(int $id): User { if (!$row) throw new NotFound(); }` força caller a usar `try/catch` para o caso esperado de "não achou". Custa stack trace por chamada, polui logs, e é mais lento que `?User` ou `User|NotFound`. Reserve exceção para o **inesperado**.

**Cenário 3 — perda de contexto em re-throw**: `catch (\Exception $e) { throw new MyException('falhou'); }` apaga o stack trace original. Sempre passe `previous`: `throw new MyException('falhou', 0, $e);`. Sem isso, o relatório de erro mostra "falhou" sem saber por quê.

**Cenário 4 — hierarquia própria de domínio**: criar `abstract class OrderException extends RuntimeException` e derivar `OrderNotFound`, `OrderAlreadyPaid` etc. permite ao caller `catch (OrderException $e)` tratar todo o subgrupo de forma genérica e específicas no mesmo bloco quando necessário.

## Exemplo de código PHP

**Hierarquia de domínio:**

```php
<?php
declare(strict_types=1);

namespace App\Domain\Order;

abstract class OrderException extends \RuntimeException {}

final class OrderNotFound extends OrderException {
    public static function withId(int $id): self {
        return new self("order {$id} not found");
    }
}

final class OrderAlreadyPaid extends OrderException {
    public function __construct(
        public readonly int $orderId,
        public readonly \DateTimeImmutable $paidAt,
    ) {
        parent::__construct("order {$orderId} already paid at {$paidAt->format('c')}");
    }
}

final class InvalidStateTransition extends OrderException {}
```

**Try/catch/finally com cleanup determinístico:**

```php
<?php
declare(strict_types=1);

function importFile(string $path): int {
    $fh = fopen($path, 'r');
    if ($fh === false) {
        throw new \RuntimeException("cannot open {$path}");
    }

    try {
        $count = 0;
        while (($row = fgetcsv($fh)) !== false) {
            processRow($row);
            $count++;
        }
        return $count;
    } catch (\Throwable $e) {
        // rewrap preservando o original
        throw new ImportFailedException("import {$path} failed", 0, $e);
    } finally {
        // SEMPRE executa: sucesso, exception, ou return acima
        fclose($fh);
    }
}
```

**Exception chaining e inspection:**

```php
<?php
declare(strict_types=1);

try {
    $repo->save($order);
} catch (\PDOException $db) {
    throw new OrderPersistenceFailure(
        "could not save order {$order->id}",
        previous: $db,
    );
}

// no boundary:
try {
    $useCase->execute();
} catch (\Throwable $e) {
    // caminha pela cadeia
    $root = $e;
    while ($prev = $root->getPrevious()) {
        $root = $prev;
    }
    $logger->error('failed', [
        'class' => $e::class,
        'root_class' => $root::class,
        'root_message' => $root->getMessage(),
        'trace' => $e->getTraceAsString(),
    ]);
}
```

**`throw` como expressão (PHP 8.0):**

```php
<?php
declare(strict_types=1);

// em null coalescing
$user = $repo->find($id) ?? throw OrderNotFound::withId($id);

// em arrow function
$validate = fn(int $n) => $n > 0 ? $n : throw new \InvalidArgumentException('n>0');

// em short-circuit
$config['key'] ?? throw new \LogicException('config.key missing');
```

**Boundary handler (entrypoint HTTP):**

```php
<?php
declare(strict_types=1);

try {
    $response = $router->dispatch($request);
} catch (\App\Domain\Order\OrderException $e) {
    // negócio esperado -> 4xx
    $response = JsonResponse::badRequest(['error' => $e->getMessage()]);
} catch (\App\Domain\AuthException $e) {
    $response = JsonResponse::forbidden();
} catch (\Throwable $e) {
    // INCLUI Error -> 5xx + alerta
    $logger->critical($e->getMessage(), ['exception' => $e]);
    $response = JsonResponse::serverError();
}

$response->send();
```

Note a ordem: específicos antes de genéricos. Catch de `\Throwable` por último, depois de tipos de domínio.

**Anti-padrão — `return` dentro de `finally`:**

```php
<?php
function trap(): int {
    try {
        throw new \RuntimeException('boom');
    } finally {
        return 42; // ENGOLE a exception, retorna 42
    }
}

trap(); // 42, sem rastro da exception
```

Nunca `return` em `finally`. É legal mas semanticamente venenoso.

**Convertendo `Error` em `Exception` (raríssimo):**

```php
<?php
declare(strict_types=1);

// só fazer em runtime de plugin/sandbox onde TypeError é "input do usuário"
try {
    $plugin->execute($input);
} catch (\TypeError $t) {
    throw new PluginContractViolation($plugin::class, previous: $t);
}
```

Em código de aplicação normal, `TypeError` é bug — deixa subir.

## Quando usar / quando não usar

- **Sempre** criar hierarquia própria de domínio (`abstract class XxxException extends RuntimeException`). Permite catch específico e genérico simultaneamente.
- **Sempre** preservar `previous` em re-throw. Sem isso, debug fica cego.
- **Sempre** usar `finally` para cleanup de recurso (file handle, lock, transaction).
- **`catch (\Throwable)`** **apenas** no boundary mais externo (entrypoint HTTP, comando CLI, worker), e mesmo assim só para logar e converter para resposta apropriada.
- **`catch (\Exception)`** quando quer evitar engolir `Error` propositalmente.
- **Não usar exception para fluxo esperado**: "não achou" → `?T` ou `T|NotFound`. "input inválido" → return de validation result. Exception para o que **não devia** acontecer.
- **Não estender `\Exception` diretamente**: escolha `LogicException` ou `RuntimeException` da SPL. A classificação ajuda o caller.
- **Não capturar para silenciar**: `catch (...) {}` vazio é dívida garantida. Pelo menos logue ou comente o motivo.

## Armadilhas comuns

- **`catch (\Exception)` não pega `Error`**: então `TypeError`, `DivisionByZeroError`, `ParseError` passam direto. Causa: `Error` é ramo paralelo, não filho de `Exception`. Para pegar tudo: `\Throwable`.
- **`finally` com `return` sobrescreve `return` do `try`** e **silencia exception pendente**. Causa: o `finally` é a última oportunidade de mudar o estado de retorno; PHP segue a semântica do Java aqui.
- **Re-throw sem `previous` apaga stack trace original**. Causa: cada `new Exception` captura um novo trace a partir de si; sem `previous`, a cadeia anterior some.
- **Ordem dos `catch` importa**: do mais específico ao mais genérico. Inverter faz o genérico capturar tudo. Causa: PHP avalia `catch` na ordem do código; primeiro match vence.
- **Multi-catch (`catch (A|B $e)`)** existe desde 7.1, mas `$e` recebe o tipo união. Você não consegue mais acessar membros específicos de A ou B sem `instanceof`. Causa: o tipo inferido é a união, não o concreto.
- **`getCode()` é livre**: cada exception escolhe o significado. SPL usa convenções inconsistentes; `PDOException::$code` é string SQLSTATE, mas a propriedade é `int|string`. Causa: contrato é fraco; padronize em sua hierarquia.
- **`ErrorException`** existe para converter `set_error_handler` em exception, **não** para herdar. Causa: confunde-se com "exceção de erro"; na verdade é wrapper de warnings/notices clássicos.
- **`UnhandledMatchError`** estende `Error`, não `Exception`. Match não exaustivo é bug, não negócio. Causa: o RFC do match decidiu que falta de cobertura é erro de programação.
- **`AssertionError`** depende de `zend.assertions`: em produção (`-1`) o `assert()` é compilado fora; em dev (`1`) lança. Causa: assertion não é mecanismo de validação de input, é debug.
- **Construtor sem `parent::__construct`** quebra `getMessage()`/`getCode()`: se você sobrescreve `__construct` e esquece de chamar o pai, propriedades ficam vazias. Causa: a base `Exception` só preenche em seu próprio construtor.
- **Exception em destrutor**: lançar em `__destruct` durante shutdown vira fatal não capturável (engine não tem mais frame para roteá-la). Causa: shutdown roda fora do contexto normal de execução.

## Links relacionados

- [[Strict-Types]]
- [[Sistema-de-Tipos]]
- [[Configuracao-PHP-INI]]
- [[Infra/Observabilidade-OWASP]]
- [[Padroes/SOLID]]
- [[OOP-Moderno/Enums]]

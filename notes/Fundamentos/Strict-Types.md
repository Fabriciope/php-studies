---
tags: [php, fundamentos, tipos]
fase: 1
status: draft
---

# Strict Types

## Conceito

`declare(strict_types=1)` é uma diretiva de compilação introduzida na RFC "Scalar Type Declarations" do PHP 7.0 (2015). Ela altera a semântica de coerção de tipos escalares em **chamadas de função** e em **valores de retorno**. Sem ela, o PHP opera em "coercive mode": ao passar `"42"` para um parâmetro `int`, o engine tenta converter via regras de cast. Com `strict_types=1`, qualquer mismatch dispara `TypeError`.

O modelo mental correto: a diretiva é um **rótulo no arquivo onde a chamada acontece**, não no arquivo onde a função foi declarada. Quando `B.php` chama `foo()` definida em `A.php`, o modo aplicado é o de `B.php`. Isso é contraintuitivo e cai em prova com frequência. A justificativa do RFC foi pragmática: bibliotecas podem manter compatibilidade com código legado (coercive) enquanto consumidores novos podem optar pelo modo estrito sem coordenação.

Comparado a TypeScript, `strict_types` é mais raso: cobre apenas escalares (`int`, `float`, `string`, `bool`) em fronteira de função. Não há "strict null checks" globais; nullable continua sendo opcional via `?T` ou `T|null`. Comparado a Java, é mais permissivo: `int` ainda aceita `float` sem casas decimais em modo coercive (regras de "lossy conversion").

A diretiva existe em uma única forma válida: `declare(strict_types=1);` como **primeira instrução** do arquivo, antes de qualquer `namespace`, `use` ou código. Posicioná-la em qualquer outro lugar dispara erro de compilação. Não há "modo intermediário" — é binário.

Internalmente, o Zend Engine usa um flag por frame de execução. Quando um parâmetro tipado recebe valor, o engine consulta esse flag. Em modo coercive, chama `zend_parse_arg_*` com fallback para conversão; em strict, dispara `TypeError` imediatamente em caso de incompatibilidade.

A diretiva afeta **apenas** tipos escalares (`int`, `float`, `string`, `bool`). Tipos de classe, `array`, `iterable`, `callable`, `object`, `mixed` e tipos especiais (`void`, `never`, `self`, `static`) **sempre** verificam estrutura, independente do modo. Confundir "strict" com "tipagem estática global" é equívoco comum: strict é uma **chave fina** sobre coerção escalar nas fronteiras de chamada, nada além.

Outra sutileza: `strict_types` impacta também o **retorno**. Se a função declara `: int` e o `return` produz `"5"`, o modo do **próprio arquivo da função** decide (porque o retorno é "saída"). Já a verificação de argumentos olha o arquivo do caller. A regra é: cada lado da fronteira é avaliado no contexto onde **opera** o valor.

## Por que importa

**Cenário 1 — validação de entrada de form**: um controller recebe `$_POST['quantity']` como string `"10"` e passa direto para `OrderService::create(int $qty)`. Em modo coercive, funciona "por acidente". O dia em que vier `"10.5"` ou `"abc"`, ou silenciosamente trunca para `10`, ou explode com `TypeError` em local errado. Com strict, o controller é forçado a converter explicitamente (`(int) $_POST['quantity']`) — o cast vira documentação do limite confiança/desconfiança.

**Cenário 2 — Value Objects de domínio**: um `Money` que aceita `int $cents` precisa que `1.0` seja **rejeitado**, não aceito como `1`. Sem strict, `new Money(99.99)` constrói um Money com 99 centavos — bug silencioso de arredondamento. Strict é a única forma sã de proteger invariantes financeiros.

**Cenário 3 — refactor com análise estática**: PHPStan/Psalm em nível alto pressupõem strict; eles avisam sobre conversões implícitas. Sem `declare(strict_types=1)`, dezenas de falsos negativos passam, porque o analisador assume que o engine vai coagir.

**Cenário 4 — APIs internas entre módulos**: módulo A expõe `chargeCustomer(int $id, int $amountCents)`. Módulo B chama com `chargeCustomer($_GET['id'], $_GET['amt'])`. Em coercive, "passa" — até `$_GET['amt']` vir `"1,99"` (vírgula europeia) e silenciosamente virar `1`. Strict forçaria o consumidor a documentar a fronteira de confiança.

**Cenário 5 — testes com dados sintéticos**: PHPUnit dataProviders frequentemente retornam strings que viram int em coercive. Teste passa, produção com input real falha. Strict alinha teste e produção.

A "dor sem isso" é: bugs que só aparecem em produção com input atípico, type juggling escondendo erros de programação, e impossibilidade de confiar nas assinaturas como contrato.

## Exemplo de código PHP

**Básico — coercive vs strict lado a lado:**

```php
<?php
// arquivo: coercive.php  (sem declare)
function add(int $a, int $b): int {
    return $a + $b;
}
var_dump(add("3", "4"));   // int(7) — coage string -> int
var_dump(add(1.9, 2.1));   // int(3) — TRUNCA float para int
```

```php
<?php
declare(strict_types=1); // arquivo: strict.php
function add(int $a, int $b): int {
    return $a + $b;
}
var_dump(add("3", "4"));   // TypeError
var_dump(add(1.9, 2.1));   // TypeError
var_dump(add(1, 2));       // int(3)
```

**Avançado — a regra do arquivo chamador:**

```php
<?php
// arquivo: lib.php  (SEM strict)
function multiply(int $a, int $b): int {
    return $a * $b;
}
```

```php
<?php
declare(strict_types=1); // arquivo: caller-strict.php
require 'lib.php';
multiply("3", 4); // TypeError — vale o modo do CALLER (strict.php)
```

```php
<?php
// arquivo: caller-coercive.php  (SEM strict)
require 'lib.php';
multiply("3", 4); // int(12) — vale o modo do CALLER
```

A declaração na `lib.php` é irrelevante para a coerção dos argumentos; só conta o arquivo de onde a chamada parte.

**Tabela de conversões em strict vs coercive:**

| Chamada                         | Coercive       | Strict      |
|---------------------------------|----------------|-------------|
| `f(int $x)` recebendo `"5"`     | aceita `5`     | TypeError   |
| `f(int $x)` recebendo `"5abc"`  | aceita `5` + warning | TypeError   |
| `f(int $x)` recebendo `5.0`     | aceita `5`     | TypeError   |
| `f(int $x)` recebendo `5.7`     | aceita `5` (trunca) | TypeError   |
| `f(float $x)` recebendo `5`     | aceita `5.0`   | **aceita `5.0`** (widening permitido) |
| `f(string $x)` recebendo `123`  | aceita `"123"` | TypeError   |
| `f(bool $x)` recebendo `1`      | aceita `true`  | TypeError   |
| `f(bool $x)` recebendo `"0"`    | aceita `false` | TypeError   |

Notar a exceção: int -> float é permitido mesmo em strict, porque é conversão sem perda.

**Anti-padrão — strict misturado a casts implícitos:**

```php
<?php
declare(strict_types=1);

function priceTag(float $price): string {
    return "R$ " . $price; // concatena float com string — PHP coage internamente
}

// strict NÃO cobre operações dentro do corpo. Só assinatura/retorno.
echo priceTag(9.9); // "R$ 9.9" — funciona, mas a concatenação foi coerção implícita
```

A strict só guarda a **fronteira** da função. Dentro, vale a aritmética PHP padrão.

**Antes e depois — corrigindo um controller:**

```php
<?php
// antes — sem strict, sem cast
function createOrder($qty, $price) { /* ... */ }
createOrder($_POST['qty'], $_POST['price']);
```

```php
<?php
declare(strict_types=1);

function createOrder(int $qty, int $priceCents): Order { /* ... */ }

// caller força a conversão e a documenta
$qty = filter_var($_POST['qty'], FILTER_VALIDATE_INT)
    ?? throw new InvalidArgumentException('qty inválido');
$priceCents = (int) round(((float) $_POST['price']) * 100);
createOrder($qty, $priceCents);
```

**Integração com PHPStan:**

```neon
# phpstan.neon
parameters:
    level: 9
    paths: [src]
    treatPhpDocTypesAsCertain: true
```

Com `strict_types` em todo arquivo e PHPStan no nível 9, conversões implícitas internas (`"R$ " . $price`) também aparecem como aviso (`strictRules.allRules`). É a única forma realista de ter "TypeScript-grade" em PHP.

**Detectando arquivos sem strict via ripgrep:**

```bash
# lista arquivos PHP sem declare(strict_types=1)
rg -L --files-without-match 'declare\(strict_types=1\)' -g '*.php' src/

# CI: falha se algum arquivo de src/ não tem strict
rg -L --files-without-match 'declare\(strict_types=1\)' -g '*.php' src/ \
  | grep . && exit 1 || exit 0
```

Adotar em CI evita regressão silenciosa. Em projetos novos, é uma regra "one-liner" no pipeline.

## Quando usar / quando não usar

- **Sempre em código novo.** Primeira linha de todo `.php`. Não há razão técnica para omitir.
- **Sempre em camada de domínio** (Entities, Value Objects, Use Cases). Invariantes precisam de garantia.
- **Sempre em libs publicadas no Packagist.** Bibliotecas devem ser estritas internamente; o **consumidor** decide o modo dele.
- **Gatilho para revisar:** ao subir versão major do framework, ative strict em arquivo por arquivo se ainda não estiver.
- **Não pular** em controllers só porque "vem string do form". O lugar de converter é o controller — explicitamente.
- **Cuidado em migração de legado:** ativar em arquivo que recebia `"1"` de form e tratava como int **vai quebrar**. Esse "quebrar" é a feature; planeje a migração arquivo por arquivo com testes.

## Armadilhas comuns

- **Posição da diretiva**: precisa ser a primeira instrução **executável** do arquivo. Pode haver comentário/shebang antes, mas nada de `<?php\n\nnamespace X;` antes do `declare`. Causa: parser exige antes de qualquer token que altere estado.
- **A regra do arquivo chamador**: já citada. Causa: o RFC permitiu adoção gradual sem quebrar libs existentes, então o modo é do call-site, não do callee.
- **`int` recebendo `1.0` quebra**: floats sem casas decimais não são "promovidos a int" em strict. Causa: a conversão float→int sempre teria potencial de perda (overflow, NAN), então o RFC proibiu.
- **`float` recebendo `int` funciona**: int sempre cabe em float (até IEEE 754 quebrar). Conversão "widening" sem perda é permitida.
- **`null` exige tipo nullable**: `function f(int $x)` rejeita `null` em strict (e em coercive a partir do PHP 8.1, agora com deprecation). Use `?int` ou `int|null`.
- **Strict não cobre operadores internos**: `$a + "5"` dentro do corpo da função ainda coage. Causa: strict é uma diretiva de **fronteira**, não um modo global da linguagem.
- **`ini_set('strict_types', ...)` não existe**: a diretiva é de compilação, não de runtime. Causa: o engine precisa decidir o modo antes de gerar opcodes da função.
- **Métodos mágicos têm regras estranhas**: `__call`, `__get`, `__set` recebem tipos via reflection; a coerção nos argumentos passados ao método **real** segue o modo do chamador. Causa: a indireção mágica não muda quem está chamando.
- **`call_user_func` propaga o modo do site da chamada**, não do arquivo onde a callback foi definida. Causa: o engine identifica o frame da chamada efetiva, não onde a closure nasceu.
- **Eval com strict_types**: `eval('declare(strict_types=1); ...')` **não funciona** — declare em eval é proibido. Causa: ambiguidade sobre escopo.

## Links relacionados

- [[Sistema-de-Tipos]]
- [[Hierarquia-Throwable]]
- [[OOP-Moderno/Readonly-Classes]]
- [[OOP-Moderno/Enums]]
- [[Padroes/SOLID]]
- [[Infra/Observabilidade-OWASP]]

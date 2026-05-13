---
tags: [php, oop, enums]
fase: 2
status: draft
---

# Enums

## Conceito

Enum em PHP 8.1+ é um **tipo fechado de primeira classe**, introduzido pela RFC "Enumerations". Internamente, um `enum` é uma classe especial cujas instâncias são **singletons** controlados pelo runtime: cada `case` declarado equivale a um objeto único, comparável por identidade (`===`). Isso é diferente de constantes de classe — `OrderStatus::Pending` é um *objeto* tipado, não uma string.

Existem duas variantes. O **enum puro** (`enum Status { case Active; }`) não tem valor escalar associado; só faz sentido como token de domínio. O **backed enum** (`enum Status: string { case Active = 'active'; }`) atribui a cada case um valor `int` ou `string`, que serve como representação serializável — útil para persistência, APIs e URLs. O backing type é fixo na declaração e todos os cases precisam respeitá-lo.

O modelo mental é parecido com `sealed class` de Kotlin ou `enum class` de Java: um conjunto fechado de valores conhecido em compile-time, com possibilidade de carregar comportamento. Diferente de Java, porém, enums em PHP **não permitem estado mutável** — você não pode declarar propriedades (exceto as constantes). Métodos são permitidos, mas costumam ser puros, geralmente implementados como `match($this)`.

Enums podem `implements` interfaces, e backed enums automaticamente implementam `BackedEnum` (que estende `UnitEnum`). Você pode usar tipos enum como type-hints em parâmetros, retornos e propriedades — o que move a validação para a borda do sistema: ou o valor entra no formato certo via `from()`/`tryFrom()`, ou nem chega ao domínio.

Comparado a Java/Kotlin, enums PHP são mais enxutos: sem construtor com argumentos, sem propriedades de instância, sem `values()` (usa-se `cases()`). Em compensação, casam perfeitamente com `match` e com strict types, formando uma das ferramentas mais robustas do PHP moderno para modelagem de domínio.

## Por que importa

**Cenário 1 — boundary do HTTP**: uma rota recebe `?status=paid`. Sem enum, a string flutua pelo código até falhar tarde, num `if` esquecido. Com `OrderStatus::from($request->query('status'))`, a validação acontece no momento da entrada: input inválido vira `ValueError` imediatamente, e do controlador para baixo o tipo é garantido.

**Cenário 2 — refator de código legado**: ao trocar `'pending'` por `'awaiting_payment'`, uma busca textual no projeto deixa pontos esquecidos. Com `OrderStatus::Pending`, o rename é mecânico via IDE; qualquer call site quebra em compile-time se o case for removido.

**Cenário 3 — exaustividade**: quando um novo status surge (`Refunded`), você quer que **todos** os `switch`/`match` pelo código sejam revisados. Usando `match` sem `default` sobre um enum, o PHP lança `UnhandledMatchError` em runtime para o case faltante; combinado com análise estática (PHPStan/Psalm), o erro vira aviso em CI.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// 1) Enum puro: tokens de domínio sem valor escalar.
enum Suit {
    case Hearts;
    case Diamonds;
    case Clubs;
    case Spades;

    public function color(): string {
        return match ($this) {
            self::Hearts, self::Diamonds => 'red',
            self::Clubs, self::Spades    => 'black',
        };
    }
}

// Comparação por identidade — cada case é singleton.
$a = Suit::Hearts;
$b = Suit::Hearts;
var_dump($a === $b); // true
```

```php
<?php
declare(strict_types=1);

// 2) Backed enum: valor escalar para serialização e persistência.
enum OrderStatus: string {
    case Pending   = 'pending';
    case Paid      = 'paid';
    case Shipped   = 'shipped';
    case Cancelled = 'cancelled';

    // Métodos: lógica fechada sobre os cases.
    public function isFinal(): bool {
        return match ($this) {
            self::Shipped, self::Cancelled => true,
            self::Pending, self::Paid      => false,
            // SEM default — adicionar case novo força revisão.
        };
    }

    public function label(): string {
        return match ($this) {
            self::Pending   => 'Aguardando pagamento',
            self::Paid      => 'Pago',
            self::Shipped   => 'Enviado',
            self::Cancelled => 'Cancelado',
        };
    }
}

// Boundary HTTP: validação automática na entrada.
$raw    = $_GET['status'] ?? '';
$status = OrderStatus::from($raw);          // lança ValueError se inválido
$maybe  = OrderStatus::tryFrom($raw);       // retorna null se inválido

// Persistência: usa o backing value, não o name.
$pdo->prepare('UPDATE orders SET status = ? WHERE id = ?')
    ->execute([$status->value, $orderId]);
```

```php
<?php
declare(strict_types=1);

// 3) Enum implementando interface — combina contrato e tipo fechado.
interface HasLabel {
    public function label(): string;
}

enum Priority: int implements HasLabel {
    case Low    = 1;
    case Medium = 2;
    case High   = 3;

    public function label(): string {
        return match ($this) {
            self::Low    => 'Baixa',
            self::Medium => 'Média',
            self::High   => 'Alta',
        };
    }

    // Métodos estáticos: factories alternativas.
    public static function fromScore(int $score): self {
        return match (true) {
            $score < 30  => self::Low,
            $score < 70  => self::Medium,
            default      => self::High,
        };
    }
}
```

```php
<?php
declare(strict_types=1);

// 4) Iterando todos os cases e serializando para JSON.
$all = OrderStatus::cases();   // array<int, OrderStatus>

$payload = array_map(
    fn(OrderStatus $s) => ['value' => $s->value, 'label' => $s->label()],
    $all,
);
echo json_encode($payload, JSON_THROW_ON_ERROR);

// json_encode de um único case backed gera apenas o backing value:
echo json_encode(OrderStatus::Paid); // "paid"
// Enum puro não é JSON-serializable diretamente: precisa de JsonSerializable.
```

```php
<?php
declare(strict_types=1);

// 5) Match exaustivo sem default — guarda contra novos cases.
function pricingFor(OrderStatus $s): string {
    return match ($s) {
        OrderStatus::Pending   => 'Pode cancelar gratuitamente',
        OrderStatus::Paid      => 'Cancelamento gera reembolso',
        OrderStatus::Shipped   => 'Cancelamento sujeito a frete',
        OrderStatus::Cancelled => 'N/A',
        // adicionar Refunded ao enum quebra aqui em runtime.
    };
}
```

## Quando usar / quando não usar

- **Use** quando o conjunto é **fechado e conhecido em compile-time**: status, papéis, dias da semana, modos de pagamento, níveis de severidade.
- **Use backed (string)** quando o enum cruza fronteiras (DB, JSON, query string) — o `value` é o que persiste.
- **Use backed (int)** quando ordenação numérica importa ou para economizar bytes (prioridade, severidade).
- **Use puro** quando o valor escalar não importa e seria arbitrário (tokens de domínio puros).
- **Não use** quando o conjunto vem de banco/config em runtime — vire Value Object com string validada ou collection de objetos.
- **Não use** como substituto de classe quando precisa de estado por instância — enum não tem propriedades de instância.
- **Não use** para flags bitwise — não há suporte nativo; modele com array de cases ou library específica.

## Armadilhas comuns

- **`from()` lança `ValueError`, não `TypeError`**: a hierarquia de exceções é diferente da type coercion comum, então um `catch (TypeError)` não captura. Isso acontece porque o valor passou no type check (era string ou int) mas falhou no mapeamento de case.
- **`match` sem `default` lança `UnhandledMatchError`**: parece bug, mas é feature. Acontece porque `match` é estrito; o ganho é forçar revisão ao adicionar cases — vale a pena, mas exige disciplina nos testes.
- **`json_encode` de enum puro retorna `{}`**: o PHP só sabe serializar `BackedEnum` automaticamente. Enums puros precisam implementar `JsonSerializable` ou ser tratados explicitamente.
- **Enum não é instanciável via `new`**: faz sentido (singletons), mas confunde quem espera padrão de classe. Frameworks que hidratam objetos via reflection precisam suporte específico.
- **`cases()` retorna na ordem de declaração**: aparenta detalhe trivial, mas listas em UI muitas vezes querem ordem alfabética ou customizada — não confie na ordem do `cases()` para apresentação.
- **`spl_object_id` em enum** retorna o mesmo id sempre para o mesmo case (singleton). Útil para entender por que `===` funciona, mas pode surpreender quem espera object identity por instância.
- **Migrar enum entre namespaces** quebra dados serializados (`unserialize`) que referenciem o FQN antigo. A serialização guarda o nome completo; renomeie com cuidado e considere strategy de versionamento.
- **Backing value duplicado é fatal error**: dois cases com o mesmo `value` impedem a classe de carregar. Detecção é em load-time, não em runtime — ótimo, mas pode pegar de surpresa em geração de código.

## Links relacionados

- [[Sistema-de-Tipos]]
- [[Readonly-Classes]]
- [[Arquitetura/Aggregates]]
- [[Arquitetura/Value-Objects]]
- [[Atributos]]
- [[Constructor-Promotion]]

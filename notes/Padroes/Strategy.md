---
tags: [php, padroes, strategy]
fase: 3
status: draft
---

# Strategy

## Conceito

Strategy é um dos 23 padrões originais do *Design Patterns* (GoF, 1994), categoria comportamental. A ideia central: **encapsular um algoritmo em uma interface e consumi-lo polimorficamente**. Trocar de algoritmo = trocar a implementação injetada, sem tocar o consumidor. Substitui `if/elseif/switch` que selecionam comportamento por tipo, eliminando a "escada de else" e libertando cada estratégia para ser testada em isolamento.

O modelo mental é "verbo como objeto". Onde antes havia `calcularImposto($pedido, $regiao)` com 10 ramos, passa a haver `CalculadoraImposto` (interface) e implementações concretas (`ICMSImposto`, `SimplesNacional`, `ImpostoFederal`). O caller deixa de saber qual ramo executa; recebe pronto pelo construtor ou parâmetro.

```
+------------+        usa        +---------------------+
|  Cliente   | ----------------> | StrategyInterface   |
+------------+                   +---------------------+
                                   ^         ^        ^
                                   |         |        |
                              ConcreteA  ConcreteB  ConcreteC
```

Comparado a padrões irmãos:

| Padrão | Foco | Diferença chave |
|---|---|---|
| **State** | Comportamento muda com estado interno do objeto | Estado **muta** internamente; objeto troca de estratégia sozinho |
| **Strategy** | Comportamento muda por escolha externa | Quem injeta decide; objeto **não muta** |
| **Command** | Encapsular operação para executar/agendar/desfazer | Tem `execute()` mas inclui contexto/parâmetros |
| **Template Method** | Esqueleto fixo, passos variáveis via herança | Inversão: pai chama filho. Strategy compõe; Template herda |
| **Specification** | Predicado de seleção | Strategy faz; Specification decide se deve fazer |

Em PHP 8.x, Strategy pode ser materializado como interface + classes, mas também como **Closure** quando a estratégia é trivial (uma função pura sem estado), ou via `match` quando o conjunto de variantes é fechado e pequeno. O critério para promover de `match` a Strategy real: surgimento de estado, dependências externas, necessidade de teste isolado, ou explosão de ramos.

## Por que importa

Strategy é o padrão de maior retorno no dia-a-dia. Toda vez que um `match($tipo)` retorna comportamentos diferentes (não apenas valores), há uma Strategy implícita esperando para nascer. Extrair libera:

- **Open/Closed**: nova estratégia = nova classe; código antigo intocado.
- **Teste**: cada estratégia testada com seus próprios inputs, sem montar contexto do `switch` inteiro.
- **Substituição em runtime**: usuário escolhe regra de frete; container injeta a impl certa.
- **Documentação por nome**: `ProgressiveTaxStrategy` diz mais que `case 'progressive': ...`.

Cenários concretos:

- **Cálculo de frete** (PAC, SEDEX, transportadora própria, retirada): cada um com regras radicalmente diferentes. Strategy resolve.
- **Política de retry** (linear, exponencial, jittered): consumidor único, política variável.
- **Serializadores** (JSON, XML, CSV): mesma interface, formatos distintos.
- **Algoritmos de matching** em busca: BM25, cosine, fuzzy.

Quando **não** aplicar: quando o que varia é só **dado**, não comportamento. Tipo de cartão (Visa/Master) raramente exige Strategy — é enum. Já gateway de pagamento (Stripe/Pagar.me/Mercado Pago) exige Strategy porque cada um tem fluxo distinto.

## Exemplo de código PHP

### 1) Interface clássica com VOs `readonly`

```php
<?php
declare(strict_types=1);

interface DiscountStrategy
{
    public function apply(Money $total): Money;
}

final readonly class PercentOff implements DiscountStrategy
{
    public function __construct(private int $percent)
    {
        if ($percent < 0 || $percent > 100) {
            throw new \InvalidArgumentException('percent out of range');
        }
    }

    public function apply(Money $total): Money
    {
        $newCents = (int) ($total->cents * (100 - $this->percent) / 100);
        return new Money($newCents, $total->currency);
    }
}

final readonly class FixedOff implements DiscountStrategy
{
    public function __construct(private Money $amount) {}

    public function apply(Money $total): Money
    {
        if ($total->currency !== $this->amount->currency) {
            throw new \DomainException('currency mismatch');
        }
        return new Money(max(0, $total->cents - $this->amount->cents), $total->currency);
    }
}

final readonly class NoDiscount implements DiscountStrategy
{
    public function apply(Money $total): Money
    {
        return $total;
    }
}

// Consumidor: agnóstico a qual estratégia recebe.
function checkout(Cart $cart, DiscountStrategy $discount): Money
{
    return $discount->apply($cart->subtotal());
}
```

### 2) Strategy via Closure (quando não há estado)

```php
<?php
declare(strict_types=1);

// Quando a estratégia é uma função pura curta, interface+classe é overhead.
// Use Closure tipada.

/** @var callable(Money): Money $strategy */
$percentOff = fn (Money $t): Money => new Money(
    (int) ($t->cents * 0.9),
    $t->currency,
);

$noDiscount = fn (Money $t): Money => $t;

function checkoutClosure(Cart $cart, \Closure $apply): Money
{
    return $apply($cart->subtotal());
}

// Promoção a interface quando:
// - aparece estado (cache, contador, config injetada);
// - aparecem variantes nomeadas que querem nome próprio;
// - precisa de teste com mock/spy.
```

### 3) Strategy parametrizada (família de estratégias)

```php
<?php
declare(strict_types=1);

// Estratégia configurada por parâmetros — útil quando o algoritmo é o mesmo
// mas com tuning distinto (tier de cliente, faixa de valor etc).

final readonly class TieredDiscount implements DiscountStrategy
{
    /** @param array<int, int> $tiers Mapa minCents => percent */
    public function __construct(private array $tiers) {}

    public function apply(Money $total): Money
    {
        $percent = 0;
        foreach ($this->tiers as $minCents => $p) {
            if ($total->cents >= $minCents && $p > $percent) {
                $percent = $p;
            }
        }
        return (new PercentOff($percent))->apply($total);
    }
}

// Configuração diferente, mesma classe:
$bronze = new TieredDiscount([10_000 => 5, 50_000 => 10]);
$gold   = new TieredDiscount([5_000 => 10, 20_000 => 15, 100_000 => 25]);
```

### 4) Promoção de `match` para Strategy

```php
<?php
declare(strict_types=1);

// Início: match é honesto e direto.
function shippingCostStart(string $method, float $weightKg): int
{
    return match ($method) {
        'pac'    => (int) (1500 + $weightKg * 200),
        'sedex'  => (int) (2500 + $weightKg * 400),
        'pickup' => 0,
    };
}

// Quando cada ramo cresce, precisa de API externa, precisa de teste isolado,
// e novos métodos surgem, promova:
interface ShippingStrategy
{
    public function cost(float $weightKg, Address $to): int;
}

final readonly class Pac implements ShippingStrategy
{
    public function __construct(private CorreiosClient $client) {}
    public function cost(float $w, Address $to): int { /* chama Correios */ return 0; }
}

final readonly class Sedex implements ShippingStrategy
{
    public function __construct(private CorreiosClient $client) {}
    public function cost(float $w, Address $to): int { /* chama Correios */ return 0; }
}

final readonly class Pickup implements ShippingStrategy
{
    public function cost(float $w, Address $to): int { return 0; }
}
```

### 5) Strategy escolhida em runtime via Factory

```php
<?php
declare(strict_types=1);

// Factory acopla a escolha; Strategy executa.
final readonly class DiscountFactory
{
    public function fromCode(string $code, ?int $arg = null): DiscountStrategy
    {
        return match ($code) {
            'percent' => new PercentOff($arg ?? throw new \InvalidArgumentException('arg required')),
            'fixed'   => new FixedOff(new Money($arg ?? 0, 'BRL')),
            'none'    => new NoDiscount(),
            default   => throw new \DomainException("unknown discount code: {$code}"),
        };
    }
}

// Controller -> factory -> strategy:
$strategy = $factory->fromCode($request->discountCode, $request->discountValue);
$final = checkout($cart, $strategy);
```

## Quando usar / quando não usar

- **Usar** quando `match`/`switch` seleciona **lógica** (não valor) por tipo, e o número de ramos cresce ou cada ramo tem dependências distintas.
- **Usar** quando a regra precisa ser configurável em runtime (admin, feature flag, A/B).
- **Usar** quando duas estratégias têm o mesmo "lugar" no fluxo mas implementações divergentes (PAC vs SEDEX).
- **Combinar com [[Factory]]** quando a escolha depende de input externo (código, config, header).
- **Não usar** com uma única estratégia e nenhuma perspectiva de segunda — interface especulativa custa sem entregar.
- **Não usar** quando o que varia é só dado: enum + propriedade resolve.
- **Não substituir** State: se o objeto **muda de estratégia internamente** ao longo do ciclo de vida, é State.

## Armadilhas comuns

- **Strategy com estado**: estratégia que mantém cache, contadores ou conexão viola substituibilidade — chamadas repetidas dão resultados diferentes. Mantenha pura ou isole estado em colaborador injetado.
- **Strategy gorda**: uma única estratégia com `if` interno por subtipo — você reintroduziu o switch. Quebre por subtipo real.
- **Confusão com Specification**: Specification responde "este item satisfaz X?" (boolean); Strategy executa "como X deve ser feito?". Specification compõe; Strategy executa.
- **Confusão com Template Method**: se o esqueleto é fixo e só passos mudam, Template Method (herança) pode ser mais direto — mas Strategy continua mais flexível por compor.
- **Estratégia que precisa do objeto inteiro**: assinatura `apply(Pedido $p)` quando só `total` interessa acopla demais. Receba só o necessário.
- **Estratégia null como "nenhum"**: usar `?DiscountStrategy` e `if ($d) {...}` reintroduz null check. Use `NoDiscount` (Null Object) e simplifique callers.
- **Hierarquia paralela a enum**: criar interface + classes para variar só uma constante. Enum com método (PHP 8.1+) resolve.
- **Strategy injetada via container quando o input é runtime**: o container resolve em wire-time; se a escolha depende do request, use Factory que recebe input e devolve strategy.

## Links relacionados

- [[Factory]]
- [[Specification]]
- [[Decorator]]
- [[Observer]]
- [[Padroes/SOLID]]
- [[Arquitetura/Use-Cases]]

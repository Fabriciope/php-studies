---
tags: [php, oop, php8]
fase: 2
status: draft
---

# Constructor Promotion + First-Class Callable

## Conceito

**Constructor Property Promotion** chegou no PHP 8.0 pela RFC "Constructor Promotion". A ideia: combinar **declaração de propriedade**, **definição de parâmetro do construtor** e **atribuição** em uma única sintaxe. O parâmetro do construtor recebe um modificador de visibilidade (`public`, `protected`, `private`, e a partir de 8.1, `readonly`) e o PHP cria a propriedade correspondente, atribuindo o valor do argumento automaticamente. Elimina o ritual `__construct + $this->x = $x` que dominava qualquer classe orientada a objetos.

**First-Class Callable Syntax** chegou no PHP 8.1 pela RFC homônima. A ideia: criar um `Closure` a partir de qualquer callable usando o argumento literal `(...)` (três pontos entre parênteses). `strlen(...)` produz a closure correspondente à função `strlen`. `$obj->method(...)` produz a closure ligada a `$obj`. `ClassName::staticMethod(...)` produz a closure do método estático. Substitui as formas legacy `'strlen'`, `[$obj, 'method']`, `[ClassName::class, 'static']` — strings mágicas que análise estática quase não conseguia rastrear.

Os dois são tratados juntos porque pertencem ao mesmo ciclo de "PHP moderno utilitário": reduzem cerimônia, melhoram tipagem estática e mudam a estética idiomática do código. Antes do PHP 8, uma classe de service com cinco dependências precisava de 5 propriedades, 5 parâmetros, 5 atribuições — 15 linhas para descrever nada. Hoje são 5 linhas. Antes, registrar `[$logger, 'info']` num pipeline confundia o IDE; hoje `$logger->info(...)` é typed e refatorável.

Promotion tem limitações por design. Não funciona com `abstract function __construct` (porque abstract não tem corpo onde acontecer a atribuição); não funciona em interfaces; não pode aparecer em `__construct` de trait que precisaria conflitar em composição. Você pode misturar parâmetros promovidos e não promovidos no mesmo construtor, e ainda adicionar corpo após as atribuições automáticas — promotions rodam primeiro, código manual depois.

First-class callable opera sobre **qualquer** callable: funções globais, métodos de instância, métodos estáticos, funções anônimas, closures já criadas, métodos mágicos `__invoke`. O resultado é sempre `Closure`, nunca `string` ou `array` — o que normaliza o tipo no consumidor: aceita `Closure`, recebe `Closure`. A closure carrega o `$this` apropriado para métodos de instância, então persiste a referência ao objeto enquanto existir.

## Por que importa

**Cenário 1 — handlers e services injetáveis**: containers DI modernos (Symfony, Laravel) usam autowire por tipo no construtor. Promotion + readonly + tipos diz tudo: "esses são meus colaboradores, são imutáveis, são injetados". Toda a infraestrutura de Clean Architecture/Hexagonal economiza dezenas de linhas por classe.

**Cenário 2 — pipelines funcionais**: `array_map`, `array_filter`, decoradores de middleware. Com first-class callable, `array_map($user->fullName(...), $users)` é typesafe, refatorável e legível. A alternativa antiga `array_map([$user, 'fullName'], $users)` é string mágica.

**Cenário 3 — testes e mocks**: spy em método de outra classe vira `$tracker->record($obj->doSomething(...))`. O `Closure` capturado pode ser invocado depois sem hack de reflexão.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// 1) Antes (PHP 7 idiomático) vs depois (8.0+).

// Antes — 15+ linhas para um service trivial.
final class CreateOrderHandlerLegacy {
    private OrderRepository $orders;
    private EventBus $bus;
    private Clock $clock;

    public function __construct(OrderRepository $orders, EventBus $bus, Clock $clock) {
        $this->orders = $orders;
        $this->bus = $bus;
        $this->clock = $clock;
    }

    public function handle(CreateOrder $cmd): OrderId { /* ... */ }
}

// Depois — promotion + readonly. Mesma semântica, sem ruído.
final readonly class CreateOrderHandler {
    public function __construct(
        private OrderRepository $orders,
        private EventBus $bus,
        private Clock $clock,
    ) {}

    public function handle(CreateOrder $cmd): OrderId {
        $order = Order::create($cmd, $this->clock->now());
        $this->orders->save($order);
        foreach ($order->pullEvents() as $event) {
            $this->bus->dispatch($event);
        }
        return $order->id;
    }
}
```

```php
<?php
declare(strict_types=1);

// 2) Promotion com defaults, atributos, e corpo adicional no construtor.
final class HttpClient {
    public function __construct(
        private readonly string $baseUrl,
        private readonly int $timeoutSeconds = 30,
        #[\SensitiveParameter] private readonly ?string $apiKey = null,
        private array $defaultHeaders = ['Accept' => 'application/json'],
    ) {
        // Promotions já atribuíram. Aqui adiciono lógica derivada.
        if ($this->apiKey !== null) {
            $this->defaultHeaders['Authorization'] = 'Bearer ' . $this->apiKey;
        }
    }
}
// Observe: `#[\SensitiveParameter]` em promoted parameter funciona.
// `$defaultHeaders` é mutável porque não recebeu readonly.
```

```php
<?php
declare(strict_types=1);

// 3) First-class callable: funções globais.
$upper = strtoupper(...);     // Closure(string $string): string
$names = array_map($upper, ['ana', 'bruno', 'carla']);
// ['ANA', 'BRUNO', 'CARLA']

// Forma legacy equivalente — string mágica, sem rastreio:
// $names = array_map('strtoupper', $names);

// Combinando com pipelines de funções compostas.
$normalize = fn(string $s) => trim(strtolower($s));
$pipeline  = [$normalize, ucfirst(...)];
$result    = array_reduce($pipeline, fn($v, $f) => $f($v), '  ANA  ');
// 'Ana'
```

```php
<?php
declare(strict_types=1);

// 4) First-class callable em métodos: instância, static, magic invoke.
final class Logger {
    public function info(string $msg): void { echo "[INFO] $msg\n"; }
    public static function staticInfo(string $msg): void { echo "[S] $msg\n"; }
    public function __invoke(string $msg): void { $this->info($msg); }
}

$log = new Logger();

$infoFn   = $log->info(...);            // Closure ligada a $log.
$staticFn = Logger::staticInfo(...);    // Closure de método estático.
$invFn    = $log(...);                  // Closure de __invoke.

$infoFn('hello');           // [INFO] hello
$staticFn('hi');            // [S] hi
$invFn('yo');               // [INFO] yo

// Útil em event bus.
final class EventBus {
    /** @var list<Closure> */ private array $listeners = [];

    public function listen(\Closure $listener): void {
        $this->listeners[] = $listener;
    }
    public function dispatch(string $msg): void {
        foreach ($this->listeners as $l) { $l($msg); }
    }
}

$bus = new EventBus();
$bus->listen($log->info(...));   // sem array mágico, sem string.
$bus->dispatch('event ok');
```

```php
<?php
declare(strict_types=1);

// 5) Promotion + asymmetric visibility + readonly em 8.4.
final class Money {
    public function __construct(
        public readonly int $cents,
        public private(set) string $currency,   // muda só internamente.
    ) {}

    public function convertTo(string $newCurrency, float $rate): self {
        // ainda imutável por valor — retorna nova instância.
        return new self((int)($this->cents * $rate), $newCurrency);
    }
}
// Promotion suporta: visibilidade, readonly, asymmetric, atributos, defaults.
// NÃO suporta: variáveis "abstratas" (sem tipo + sem default em interface).
```

## Quando usar / quando não usar

- **Use promotion sempre** que o construtor só atribui parâmetros a propriedades — caso 90% das vezes em handlers, services, repositórios, VOs.
- **Combine com `readonly`** (PHP 8.1+) para classes injetáveis imutáveis — padrão canônico de DI.
- **Combine com asymmetric** (PHP 8.4+) para entidades com estado controlado.
- **Não use promotion** se o construtor tem lógica não trivial **entre** parâmetros (validação cruzada, normalização que muda valores) — promotion atribui antes do corpo; nesses casos um construtor manual deixa a ordem clara.
- **Não use promotion em interfaces** — não é suportado, e interfaces declaram contrato, não implementação.
- **Use first-class callable** sempre que substituir string/array callable — refatoração, autocompletar e análise estática agradecem.
- **Não use first-class callable** se você só precisa de inline closure (`fn($x) => ...`) sem referenciar função existente — fica indireto à toa.

## Armadilhas comuns

- **Lógica no corpo do construtor roda depois das atribuições promovidas**: se você precisava validar antes de atribuir, esse padrão não funciona. Acontece porque promotions são `$this->x = $x` injetados no início do corpo; sua validação manual aparece depois.
- **Promotion exige visibilidade explícita**: parâmetro sem `public/protected/private` não vira propriedade. Lógico, mas confunde quem espera "tudo na assinatura vira campo".
- **`readonly` em promotion exige PHP 8.1+**: em 8.0 a promotion existe mas sem readonly. Compatibilidade quebra silenciosamente em servidor antigo.
- **`$obj->method(...)` mantém referência forte a `$obj`**: a closure carrega `$this`. Em event buses de longa vida ou caches, isso impede o garbage collector de liberar o objeto. Bug clássico de "memory leak por listener esquecido".
- **First-class callable de método mágico `__call` não funciona**: o PHP precisa do método existir concretamente para criar a closure. Métodos resolvidos via `__call` não são "first-class" no sentido literal.
- **Promotion e `parent::__construct`**: se a parent tem construtor não trivial, a child precisa decidir quando chamar `parent::__construct(...)`. Promotions da child atribuem antes; chame `parent` no corpo manualmente.
- **Promoção em classe abstrata é parcial**: você pode promover, mas não pode marcar o `__construct` da abstract como `abstract` em si. O método precisa ter corpo (mesmo vazio).
- **Atributos em parâmetros promovidos aplicam à propriedade e ao parâmetro**: `#[Validate('email')]` num promoted param é lido tanto via `ReflectionParameter` quanto `ReflectionProperty`. Pode ser desejado ou confundir. Verifique `getAttributes()` no nível certo.

## Links relacionados

- [[Readonly-Classes]]
- [[Asymmetric-Visibility]]
- [[Property-Hooks]]
- [[Atributos]]
- [[Padroes/SOLID]]
- [[Sistema-de-Tipos]]

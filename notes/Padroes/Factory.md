---
tags: [php, padroes, factory]
fase: 3
status: draft
---

# Factory

## Conceito

"Factory" é um guarda-chuva para um conjunto de técnicas que **centralizam a decisão de qual implementação criar e como construí-la**. O termo, no GoF, se desdobra em variantes:

- **Simple Factory** (idiomática, não-GoF): uma classe/método que recebe input e devolve a instância concreta. Útil para escolha controlada por dado.
- **Factory Method** (GoF criacional): método abstrato (ou virtual) que subclasses sobrescrevem para escolher o tipo a criar. Inversão da decisão para a hierarquia.
- **Abstract Factory** (GoF criacional): família de fábricas que produzem famílias de objetos relacionados (`UiFactory` que cria `Button`, `Window`, `Menu` compatíveis com um tema).
- **Static Factory Method / Named Constructor**: método estático na própria classe (`Money::brl(19.90)`) que expressa intenção de criação melhor que o construtor cru.

O modelo mental: o construtor é "como nascer da própria essência" — recebe os atributos brutos. Quando o objeto pode nascer de múltiplas fontes (carrinho, dump, evento, JSON externo, formulário, importação CSV), espalhar essa tradução pelos callers vaza detalhe e duplica regra. A Factory concentra.

```
                      +---------------------+
                      |  DiscountFactory    |
                      |  ::from(input)      |
                      +----------+----------+
                                 |
            +--------------------+--------------------+
            |                    |                    |
      PercentOff             FixedOff              NoDiscount
```

Comparado a padrões irmãos:

| Padrão | Foco | Diferença chave |
|---|---|---|
| **Builder** | Construção passo-a-passo de objeto complexo | Acumula configuração antes de `build()`; útil quando o objeto tem muitos parâmetros opcionais |
| **Prototype** | Criar clonando | Fonte é instância existente; útil quando montagem é cara |
| **Factory** | Decidir o que criar e como | Foco na **escolha de tipo** e tradução de input |
| **DI Container** | Resolver grafo completo de dependências | Container resolve em wire-time; Factory cria por **contexto runtime** |

Quando combinar com **DI Container**: o container injeta a Factory; a Factory recebe input do request e cria a instância concreta. O container não conhece o input do usuário; a factory traduz isso.

## Por que importa

Construtores são para "nascer da própria essência". Quando há múltiplos caminhos de origem, ou quando a escolha de subtipo depende de input, espalhar `new` pelos callers gera:

- **Duplicação**: cada caller que mapeia string→subtipo reimplementa a tabela.
- **Acoplamento**: callers passam a conhecer **todas** as implementações concretas.
- **Construtores poluídos**: parâmetros opcionais demais, sentinelas `null`, lógica de validação em quem só queria criar.

Cenários:

- **Hidratação de evento**: receber payload de webhook e criar o `Event` certo conforme `type`. Sem Factory, o handler conhece todas as classes.
- **Importação CSV**: cada linha vira `Order`, `Customer` ou `Product` conforme prefixo. Factory mapeia.
- **Money/VOs**: `new Money(1990, 'BRL')` é tecnicamente correto mas pobre. `Money::brl(19.90)` é intenção.

## Exemplo de código PHP

### 1) Named Constructors (Static Factory Methods)

```php
<?php
declare(strict_types=1);

final readonly class Money
{
    public function __construct(
        public int $cents,
        public string $currency,
    ) {
        if ($cents < 0) {
            throw new \DomainException('cents must be non-negative');
        }
        if (strlen($currency) !== 3) {
            throw new \DomainException('currency must be ISO 4217');
        }
    }

    // Cada named constructor expressa uma intenção de origem.
    public static function brl(float $reais): self
    {
        return new self((int) round($reais * 100), 'BRL');
    }

    public static function usd(float $dollars): self
    {
        return new self((int) round($dollars * 100), 'USD');
    }

    public static function fromString(string $iso): self
    {
        // "BRL 19.90" -> Money
        if (!preg_match('/^([A-Z]{3})\s+(\d+(?:\.\d{1,2})?)$/', $iso, $m)) {
            throw new \InvalidArgumentException("invalid format: {$iso}");
        }
        return new self((int) round(((float) $m[2]) * 100), $m[1]);
    }

    public static function zero(string $currency): self
    {
        return new self(0, $currency);
    }
}

// Uso fluente e legível:
$preco = Money::brl(19.90);
$total = Money::fromString('USD 100.00');
$grátis = Money::zero('BRL');
```

### 2) Simple Factory: escolha por código

```php
<?php
declare(strict_types=1);

final readonly class DiscountFactory
{
    public function from(string $code, int|float|null $arg = null): DiscountStrategy
    {
        return match ($code) {
            'percent' => new PercentOff(
                (int) ($arg ?? throw new \InvalidArgumentException('arg required for percent')),
            ),
            'fixed' => new FixedOff(
                Money::brl((float) ($arg ?? 0)),
            ),
            'none' => new NoDiscount(),
            default => throw new \DomainException("unknown discount code: {$code}"),
        };
    }
}

// Controller usa a factory; não precisa importar implementações concretas.
$strategy = $factory->from($request->discountCode, $request->discountValue);
```

### 3) Factory Method (subclasses decidem)

```php
<?php
declare(strict_types=1);

// O esqueleto fica no pai; a decisão de qual produto criar fica no filho.
abstract class ReportGenerator
{
    final public function run(\DateTimeImmutable $period): string
    {
        $data = $this->collect($period);
        $formatter = $this->createFormatter();  // factory method
        return $formatter->render($data);
    }

    abstract protected function createFormatter(): ReportFormatter;

    /** @return array<string, mixed> */
    abstract protected function collect(\DateTimeImmutable $period): array;
}

final class PdfSalesReport extends ReportGenerator
{
    protected function createFormatter(): ReportFormatter
    {
        return new PdfFormatter();
    }

    protected function collect(\DateTimeImmutable $period): array { /* ... */ return []; }
}

final class HtmlSalesReport extends ReportGenerator
{
    protected function createFormatter(): ReportFormatter
    {
        return new HtmlFormatter();
    }

    protected function collect(\DateTimeImmutable $period): array { /* ... */ return []; }
}
```

### 4) Abstract Factory (família de produtos)

```php
<?php
declare(strict_types=1);

// Cria famílias coerentes. Útil em multi-tenant, multi-tema, multi-driver.

interface StorageFactory
{
    public function createUserRepository(): UserRepository;
    public function createOrderRepository(): OrderRepository;
    public function createSessionStore(): SessionStore;
}

final readonly class MySqlStorageFactory implements StorageFactory
{
    public function __construct(private \PDO $pdo) {}

    public function createUserRepository(): UserRepository
    {
        return new PdoUserRepository($this->pdo);
    }

    public function createOrderRepository(): OrderRepository
    {
        return new PdoOrderRepository($this->pdo);
    }

    public function createSessionStore(): SessionStore
    {
        return new PdoSessionStore($this->pdo);
    }
}

final class InMemoryStorageFactory implements StorageFactory
{
    public function createUserRepository(): UserRepository { return new InMemoryUserRepository(); }
    public function createOrderRepository(): OrderRepository { return new InMemoryOrderRepository(); }
    public function createSessionStore(): SessionStore { return new InMemorySessionStore(); }
}

// Testes usam InMemoryStorageFactory; produção usa MySqlStorageFactory.
```

### 5) Factory + DI Container

```php
<?php
declare(strict_types=1);

// O container injeta a factory; ela traduz input em instância.
// O container NÃO sabe sobre o input do request — a factory sim.

final readonly class CheckoutController
{
    public function __construct(
        private DiscountFactory $discountFactory,
        private CheckoutService $checkout,
    ) {}

    public function __invoke(CheckoutRequest $req): CheckoutResponse
    {
        $discount = $this->discountFactory->from($req->discountCode, $req->discountValue);
        return $this->checkout->run($req->cart, $discount);
    }
}
```

## Quando usar / quando não usar

- **Named constructors**: sempre que houver múltiplas formas legítimas de criar um VO ou entidade (`Money::brl`, `User::fromCsv`, `OrderId::generate`).
- **Simple Factory**: quando a escolha depende de input runtime e há 3+ alternativas.
- **Factory Method**: quando o esqueleto do fluxo é fixo mas o tipo a criar pertence à subclasse — útil em frameworks de relatório, parsers, jobs.
- **Abstract Factory**: quando o sistema tem múltiplos "perfis" coerentes (driver, tenant, tema) e produtos têm que vir do mesmo perfil.
- **Não usar** quando o construtor já é claro e direto em um único lugar — `new Foo($a, $b)` não pede factory.
- **Não confundir com DI Container**: container resolve grafo estático; factory cria por contexto dinâmico.
- **Não usar Abstract Factory** se há apenas um perfil real — interface especulativa.

## Armadilhas comuns

- **Factory virar god class**: cresce até saber criar tudo. Quebre por aggregate ou contexto de domínio (cada bounded context tem sua factory).
- **Construtor `private` quebrando hidratação por ORM/reflection**: alguns ORMs antigos exigem construtor público. Em geral, manter `public` e expor named constructors paralelos resolve. Para Doctrine, manter construtor compatível com a estratégia de hidratação.
- **Factory que faz I/O**: factory chamando banco, HTTP, FS dentro do método de criação. Vira responsabilidade dupla (criar + carregar). Separe: Repository carrega, Factory monta.
- **Switch da Factory replicado em outros lugares**: se o `match` interno da factory aparece também no Controller ou no Use Case, a abstração vazou. Centralize.
- **Esquecer validação no construtor "raw"**: named constructors validam, mas o construtor primário fica aberto e permite estado inválido. Coloque guardas no construtor.
- **Factory devolvendo `null` em caso desconhecido**: força caller a checar. Prefira lançar `DomainException` com mensagem clara — falhe rápido.
- **Confusão com Builder**: se há 7 parâmetros opcionais e o objeto precisa ser montado em etapas, é Builder, não Factory.
- **Factory que depende do container**: factory recebendo `ContainerInterface` para resolver subtipos. Service Locator disfarçado. Injete as dependências reais.

## Links relacionados

- [[Strategy]]
- [[Repository]]
- [[Decorator]]
- [[Specification]]
- [[Padroes/SOLID]]
- [[Arquitetura/Aggregates]]

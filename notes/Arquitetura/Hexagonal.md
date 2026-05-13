---
tags: [php, arquitetura, hexagonal, ports-adapters]
fase: 4
status: draft
---

# Hexagonal (Ports & Adapters)

## Conceito

A arquitetura *Hexagonal* foi formulada por **Alistair Cockburn** em 2005 sob o título original "*Ports and Adapters*". O hexágono é apenas convenção visual — não há nada de mágico no número seis. O que importa é a estrutura: um **núcleo de aplicação** rodeado por **portas** (interfaces declaradas pelo núcleo) que são preenchidas por **adapters** concretos no exterior.

A motivação original de Cockburn foi explicitar um problema recorrente: misturar **lógica de negócio** com **lógica de apresentação** (HTML, HTTP, CLI) ou de **infraestrutura** (banco, e-mail, fila) gera código impossível de testar sem orquestrar o ambiente inteiro. Hexagonal propõe: o núcleo declara contratos abstratos do que precisa; quem chama (driving) e o que ele chama (driven) são intercambiáveis.

```
                  Driving (entrada)                Driven (saída)
                                                                         
        ┌──────────────┐   ┌──────────────────┐   ┌──────────────────┐  
        │ HTTP Adapter │──▶│                  │──▶│ PDO Adapter      │  
        └──────────────┘   │                  │   └──────────────────┘  
        ┌──────────────┐   │      Núcleo      │   ┌──────────────────┐  
        │ CLI Adapter  │──▶│  (Application +  │──▶│ Stripe Adapter   │  
        └──────────────┘   │    Domain)       │   └──────────────────┘  
        ┌──────────────┐   │                  │   ┌──────────────────┐  
        │ Queue Worker │──▶│                  │──▶│ Mailer Adapter   │  
        └──────────────┘   └──────────────────┘   └──────────────────┘  
                            ▲                  ▲                        
                            │                  │                        
                       Primary Ports      Secondary Ports               
                      (interfaces que   (interfaces que o               
                       o mundo chama)    núcleo chama)                  
```

Duas categorias de porta:

- **Primary Ports (driving / entrada).** O contrato pelo qual o mundo aciona o núcleo. Em PHP é tipicamente a assinatura do *Use Case* (`PayOrderHandler::__invoke(PayOrderCommand)`). Quem dirige a aplicação. Os adapters de entrada (controllers HTTP, comandos CLI, workers) usam esse contrato.
- **Secondary Ports (driven / saída).** O contrato pelo qual o núcleo chama o mundo. `OrderRepository`, `PaymentGateway`, `EventBus`, `Mailer`. Interfaces **declaradas no Domain ou Application**, implementadas na Infrastructure.

O mecanismo técnico que sustenta isso é a *Dependency Inversion* (D do SOLID). Em vez de o Use Case importar `PDO`, ele importa `OrderRepository` (uma interface que ele mesmo declara). Em runtime, o container injeta `PdoOrderRepository`. **Em compile-time, o núcleo não conhece PDO.**

Outra distinção importante: **Hexagonal não é o mesmo que "camadas"**. Camadas são uma organização vertical (do mais externo ao mais interno). Hexagonal é uma afirmação sobre **direção de dependências** — independente de como você organiza pastas. Os dois combinam (e nesta linha de notas combinamos), mas conceitualmente são ortogonais.

## Por que importa

O retorno prático é difícil de exagerar quando o sistema dura mais de um ano:

1. **Testabilidade real do núcleo.** Com `InMemoryOrderRepository` (uma classe de 30 linhas), você testa o Use Case `PayOrder` inteiro sem subir MySQL, sem migrations, sem fixtures. Um teste roda em 2ms. Mil testes rodam em 2s. A pirâmide de testes para de ser ficção.

2. **Troca de adapter sem mexer no núcleo.** Migrar de Stripe para Adyen, de MySQL para PostgreSQL, de RabbitMQ para Kafka — cada um vira uma classe nova implementando a porta existente. Zero alteração em Domain/Application. Em 2-3 anos toda empresa enfrenta pelo menos uma dessas trocas.

3. **Reuso entre canais de entrada.** O mesmo Use Case `CancelOrder` é chamado pelo controller HTTP do app cliente, pelo CLI do suporte, pelo worker que processa webhook do gateway, e pela rotina diária de limpeza. Quatro adapters, um Use Case. Sem hexagonal, lógica copiada quatro vezes e divergindo.

4. **Bloqueio explícito a vazamentos.** A porta funciona como uma cerca: tudo que entra/sai do núcleo passa por uma interface declarada por você. É documentação executável das fronteiras.

O custo: cerimônia adicional. Cada operação contra mundo externo precisa de uma porta. Em CRUD trivial isso vira teatro vazio.

## Exemplo de código PHP

Cenário completo: caso de uso `PayOrder`, com porta de saída `PaymentGateway`, adapter Stripe, adapter HTTP de entrada, e teste com adapters in-memory.

### 1. Porta secundária declarada no Domain

```php
<?php
declare(strict_types=1);

namespace App\Domain\Payment;

use App\Domain\Order\OrderId;
use App\Domain\Shared\Money;

// Porta enxuta: o núcleo só pede o que precisa
interface PaymentGateway
{
    /**
     * @throws PaymentDeclined
     * @throws PaymentGatewayUnavailable
     */
    public function charge(OrderId $orderId, Money $amount): PaymentResult;
}

final readonly class PaymentResult
{
    public function __construct(
        public bool $approved,
        public string $providerTransactionId,
    ) {}
}
```

### 2. Use Case (porta primária) orquestrando o núcleo

```php
<?php
declare(strict_types=1);

namespace App\Application\Order\Pay;

use App\Domain\Order\{OrderNotFound, OrderRepository};
use App\Domain\Payment\PaymentGateway;
use App\Application\Shared\EventBus;

final readonly class PayOrderHandler
{
    public function __construct(
        private OrderRepository $orders,
        private PaymentGateway $gateway,
        private EventBus $bus,
    ) {}

    public function __invoke(PayOrderCommand $cmd): void
    {
        $order = $this->orders->ofId($cmd->orderId)
            ?? throw OrderNotFound::withId($cmd->orderId);

        // Adapter externo é apenas uma dependência abstrata aqui
        $result = $this->gateway->charge($order->id, $order->total());

        if (!$result->approved) {
            throw new \DomainException('payment declined');
        }

        $order->markPaid($result->providerTransactionId);
        $this->orders->save($order);

        foreach ($order->pullEvents() as $event) {
            $this->bus->dispatch($event);
        }
    }
}
```

### 3. Adapter secundário: implementação Stripe da porta

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Payment\Stripe;

use App\Domain\Payment\{PaymentGateway, PaymentResult, PaymentDeclined};
use App\Domain\Order\OrderId;
use App\Domain\Shared\Money;
use Stripe\StripeClient;

// Detalhe técnico vive aqui — fora do núcleo
final readonly class StripePaymentGateway implements PaymentGateway
{
    public function __construct(private StripeClient $stripe) {}

    public function charge(OrderId $orderId, Money $amount): PaymentResult
    {
        try {
            $charge = $this->stripe->charges->create([
                'amount'   => $amount->cents,
                'currency' => strtolower($amount->currency),
                'metadata' => ['order_id' => $orderId->value],
            ]);
        } catch (\Stripe\Exception\CardException $e) {
            throw new PaymentDeclined($e->getMessage());
        }

        return new PaymentResult(
            approved: $charge->status === 'succeeded',
            providerTransactionId: $charge->id,
        );
    }
}
```

### 4. Adapter primário: controller HTTP

```php
<?php
declare(strict_types=1);

namespace App\Presentation\Http;

use App\Application\Order\Pay\{PayOrderCommand, PayOrderHandler};
use App\Domain\Order\OrderId;
use Psr\Http\Message\{ServerRequestInterface, ResponseInterface};

final readonly class PayOrderController
{
    public function __construct(private PayOrderHandler $handler) {}

    public function __invoke(ServerRequestInterface $req): ResponseInterface
    {
        $body = json_decode((string) $req->getBody(), true, flags: JSON_THROW_ON_ERROR);

        // Traduz HTTP -> Command (vocabulário do núcleo)
        $cmd = new PayOrderCommand(orderId: new OrderId($body['order_id']));

        $this->handler->__invoke($cmd);

        return new \Nyholm\Psr7\Response(204);
    }
}
```

### 5. Teste do núcleo SEM framework, SEM banco, SEM rede

```php
<?php
declare(strict_types=1);

namespace App\Tests\Application\Order;

use App\Application\Order\Pay\{PayOrderCommand, PayOrderHandler};
use App\Domain\Order\{Order, OrderId, OrderRepository};
use App\Domain\Payment\{PaymentGateway, PaymentResult};
use App\Domain\Shared\Money;
use App\Application\Shared\EventBus;
use PHPUnit\Framework\TestCase;

final class PayOrderHandlerTest extends TestCase
{
    public function test_marks_order_paid_when_gateway_approves(): void
    {
        $orderId = new OrderId('ord_1');
        $order = Order::pending($orderId, new Money(10000, 'BRL'));

        // Adapters fake — totalmente em memória
        $repo = new InMemoryOrderRepository([$order]);
        $gateway = new class implements PaymentGateway {
            public function charge(OrderId $id, Money $amount): PaymentResult {
                return new PaymentResult(approved: true, providerTransactionId: 'tx_42');
            }
        };
        $bus = new InMemoryEventBus();

        $handler = new PayOrderHandler($repo, $gateway, $bus);
        $handler(new PayOrderCommand($orderId));

        self::assertTrue($repo->ofId($orderId)->isPaid());
        self::assertCount(1, $bus->dispatched);
    }
}
```

Esse teste roda em milissegundos. Nenhum mock framework, nenhum singleton, nenhum reset de banco entre testes.

## Quando usar / quando não usar

- **Use** em sistemas com lógica rica e ciclo de vida estendido.
- **Use** sempre que houver ≥ 2 fontes de entrada (HTTP + worker + CLI). Hexagonal força reuso natural.
- **Use** quando integrar com ≥ 1 sistema externo crítico (gateway de pagamento, ERP, e-mail transacional). Isolar via porta é seguro contra mudanças de API externa.
- **Não use** em CRUD simples (admin de cadastros). Cerimônia sem retorno.
- **Não use** em código de descarte (POC, script de migração de dados que rodará uma vez).
- **Cuidado** com hexagonal *cosmético*: pastas Domain/Application/Infra mas com SQL no Use Case e `Request` no agregado. É pior do que monólito honesto.

## Armadilhas comuns

- **Confundir hexagonal com camadas.** Hexagonal é direção de dependência; camadas são organização. Combinam mas não são sinônimo. Um pode existir sem o outro.
- **Adapter "leaky".** Método na interface que existe só porque um provider exige (`setApiVersion`, `setRetryStrategy`). Mantenha a porta enxuta — particularidade fica no adapter, configurada via DI.
- **Núcleo importando framework.** O erro mais comum. Importar `Symfony\Bundle\FrameworkBundle\Controller\AbstractController` ou `Illuminate\Http\Request` em qualquer classe de Application/Domain. Bloqueie via Deptrac/PHPStan.
- **Porta retornando tipo de framework.** `Mailer::send()` retornando `Symfony\Component\Mime\Email`. O retorno vaza o detalhe. Retorne tipos próprios.
- **Porta com método CRUD genérico.** `Repository::findBy(array $criteria)` quebra encapsulamento — qualquer chamador pode pedir qualquer combinação. Prefira métodos com intenção: `pendingOlderThan(\DateTimeImmutable)`.
- **Esquecer da porta de tempo.** `new \DateTimeImmutable()` espalhado pelo núcleo torna teste temporal um pesadelo. Injete `Clock` como porta.
- **Over-engineering em domínio pobre.** Hexagonal em CRUD trivial vira teatro. Se Use Case = "salvar form no banco", você não precisa de hexagonal — precisa de form + ORM honestos.
- **Container "mágico" no núcleo.** Use Case fazendo `Container::get('mailer')`. Quebra Dependency Injection explícita e amarra ao container. Constructor injection sempre.

## Links relacionados

- [[Camadas-DDD]]
- [[Use-Cases]]
- [[Padroes/Repository]]
- [[Aggregates]]
- [[Domain-Events-CQRS]]

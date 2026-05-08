---
tags: [php, arquitetura, hexagonal, ports-adapters]
fase: 4
status: stub
---

# Hexagonal (Ports & Adapters)

## Conceito

A aplicação tem um **núcleo** (domínio + aplicação) cercado por **portas** (interfaces que ele declara). O mundo externo se conecta via **adapters**:

- **Driving adapters** (entrada): HTTP controller, CLI, listener de fila → chamam o núcleo.
- **Driven adapters** (saída): repositório SQL, gateway HTTP, mailer → implementam portas que o núcleo declarou.

## Por que importa

Hexagonal é o que dá **testabilidade real** ao domínio: você troca o adapter de banco por in-memory e testa o fluxo inteiro sem subir MySQL. Também é o que permite trocar Stripe por PagSeguro mexendo apenas em uma classe.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// === núcleo: porta de saída declarada PELO domínio ===
namespace App\Domain\Payment;

interface PaymentGateway {
    public function charge(OrderId $id, Money $amount): PaymentResult;
}

final readonly class PaymentResult {
    public function __construct(public bool $approved, public string $providerId) {}
}

// === núcleo: use case orquestrando ===
namespace App\Application\Order;

final readonly class PayOrder {
    public function __construct(
        private OrderRepository $orders,
        private PaymentGateway $gateway,
        private EventBus $bus,
    ) {}

    public function __invoke(PayOrderCommand $cmd): void {
        $order = $this->orders->ofId($cmd->orderId)
            ?? throw OrderNotFound::withId($cmd->orderId);
        $result = $this->gateway->charge($order->id, $order->total);
        if (!$result->approved) throw new PaymentDeclined();
        $order->markPaid($result->providerId);
        $this->orders->save($order);
        foreach ($order->pullEvents() as $e) $this->bus->dispatch($e);
    }
}

// === infra: adapter de saída ===
namespace App\Infrastructure\Payment;

final readonly class StripePaymentGateway implements PaymentGateway {
    public function __construct(private \Stripe\StripeClient $stripe) {}
    public function charge(OrderId $id, Money $amount): PaymentResult { /* ... */ }
}

// === presentation: adapter de entrada ===
namespace App\Presentation\Http;

final readonly class PayOrderController {
    public function __construct(private PayOrder $useCase) {}
    public function __invoke(ServerRequestInterface $req): ResponseInterface {
        $cmd = PayOrderCommand::fromRequest($req);
        $this->useCase->__invoke($cmd);
        return new Response(204);
    }
}
```

## Quando usar / quando não usar

- **Sistemas com lógica de negócio rica** que vai durar anos.
- **Quando há ≥2 fontes de entrada** (HTTP + CLI + webhook + worker) — hexagonal força reuso.
- **Não usar** em CRUD onde a "regra" é só persistir o que vem do form. Vira teatro.

## Armadilhas comuns

- Achar que hexagonal = camadas. Hexagonal é sobre **direção das dependências** (o núcleo declara o que precisa); camadas é sobre **organização**. Combinam, mas não são sinônimo.
- Adapter "leaky": método na interface que existe só porque um provider exige (`StripeGateway::setApiVersion()` na porta). Mantenha porta enxuta; particularidade fica no adapter.
- Núcleo importando classes do framework — é o erro mais comum. Bloqueie via Deptrac/PHPStan.

## Links relacionados

- [[Camadas-DDD]]
- [[Use-Cases]]
- [[Padroes/Repository]]

---
tags: [php, arquitetura, use-cases, application-service]
fase: 4
status: stub
---

# Use Cases (Application Services)

## Conceito

Cada Use Case = uma classe = uma intenção do usuário/sistema. Recebe um **Command** (DTO de entrada), orquestra agregados e portas, devolve um **Result** (ou `void`). Sem regra de negócio — só coreografia.

## Por que importa

Espalhar fluxo em controllers gigantes acopla HTTP a domínio. Um Use Case por arquivo torna o sistema **legível pela ementa** — `ls Application/` lista o que o sistema faz.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// command: VO imutável de entrada
final readonly class CancelOrderCommand {
    public function __construct(
        public OrderId $orderId,
        public string $reason,
        public UserId $cancelledBy,
    ) {}
}

// use case: SRP rigoroso
final readonly class CancelOrder {
    public function __construct(
        private OrderRepository $orders,
        private EventBus $bus,
        private Clock $clock,
    ) {}

    public function __invoke(CancelOrderCommand $cmd): void {
        $order = $this->orders->ofId($cmd->orderId)
            ?? throw OrderNotFound::withId($cmd->orderId);

        $order->cancel($cmd->reason, $cmd->cancelledBy, $this->clock->now());

        $this->orders->save($order);
        foreach ($order->pullEvents() as $e) {
            $this->bus->dispatch($e);
        }
    }
}
```

## Quando usar / quando não usar

- **Sempre** em hexagonal: é o ponto de entrada do núcleo.
- **Um use case = um arquivo = uma transação**. Use case orquestrando outro use case é cheiro de domínio mal modelado.
- **CRUD trivial** pode pular — mas ganhe consistência: ou todos os fluxos passam por Use Case, ou nenhum.

## Armadilhas comuns

- Use Case carregando regra de negócio (`if ($order->total > 1000) ...`). Regra mora no **agregado**, não aqui.
- Command com tipos primitivos (`string $orderId`) — perde força do sistema de tipos. Use VOs (`OrderId`).
- Retornar entidade do domínio direto pra controller — vaza modelo. Devolva DTO/Read Model.
- Usar Use Case como "service" de leitura. Leituras pesadas vão por **query / read model**, não orquestração.

## Links relacionados

- [[Camadas-DDD]]
- [[Hexagonal]]
- [[Aggregates]]
- [[APIs/REST-Maduro]]

---
tags: [php, arquitetura, use-cases, application-service]
fase: 4
status: draft
---

# Use Cases (Application Services)

## Conceito

O termo **Application Service** vem de Eric Evans (*DDD*, 2003): "uma fina camada que coordena tarefas e delega trabalho a objetos do domínio. Não tem estado nem regra de negócio". O termo **Use Case** vem da *Clean Architecture* de Robert C. Martin (2012), inspirado por Ivar Jacobson (anos 90, casos de uso UML). Para fins práticos modernos, no PHP, **são a mesma coisa**: uma classe por intenção do usuário/sistema, com um método público que executa.

O modelo mental: cada Use Case é a **resposta a uma pergunta do tipo "o sistema faz X?"**. Se o produto diz "o cliente pode cancelar um pedido", existe a classe `CancelOrderHandler`. Se diz "o admin pode emitir reembolso parcial", existe `IssuePartialRefundHandler`. Listar `src/Application/` é listar **a ementa funcional do sistema**.

```
       Adapter de entrada              Núcleo                 Adapter de saída
       ┌──────────────┐    Command   ┌──────────┐  Portas   ┌─────────────────┐
       │  HTTP / CLI  │─────────────▶│ Use Case │──────────▶│ Repo / Gateway  │
       │   / Worker   │              │ Handler  │           │ EventBus / Clock│
       └──────────────┘    Result    └──────────┘           └─────────────────┘
                       ◀─────────────       │
                                            ▼
                                   ┌─────────────────┐
                                   │  Aggregate Root │  ◀── regra de negócio
                                   └─────────────────┘
```

Estrutura típica em três peças por caso de uso:

1. **Command (ou Query)** — DTO de entrada, imutável (`readonly`), com VOs no lugar de primitivas.
2. **Handler** — a classe do Use Case. Único método público (geralmente `__invoke` ou `handle`).
3. **Result** — opcional. DTO de saída quando o caller precisa de algo além de "ok/erro". Não é entidade do domínio; é projeção.

**Command/Query split** (origem em CQRS, mas válido fora dele): Use Cases que **modificam estado** são *Commands* (verbo no infinitivo: `PayOrder`, `CancelSubscription`); Use Cases que **só leem** são *Queries* (`ListPendingOrders`, `GetCustomerProfile`). A separação documenta intenção e evita "Use Case híbrido" que lê e escreve.

Em Clean Architecture aparece também a noção de **Input Port** e **Output Port**: input é a assinatura do Use Case (o que o controller chama); output é a interface usada para devolver resultado (ex: `Presenter`). Em PHP a maioria dos projetos colapsa isso retornando o Result diretamente — formal demais é raro pagar.

Sobre **transações**: o Use Case é a **unidade transacional natural**. `beginTransaction` no início, `commit` no fim, `rollback` em exceção. Isso é responsabilidade do Use Case (ou de um decorator que envolve todos os Use Cases) — **não** do controller, **não** do agregado. Decorator transacional é o padrão limpo (ver exemplo).

Sobre **idempotência**: Use Cases que mutam estado devem ser idempotentes quando expostos via HTTP/fila. A técnica padrão é exigir um `idempotencyKey` no Command, gravar a chave + resultado numa tabela, e retornar o resultado anterior em retentativas. Sem isso, retry de cliente/worker duplica operação.

Sobre **composição**: a regra é "**Use Case não chama Use Case**". Se você precisa, a regra compartilhada vira *Domain Service* (objeto do domínio que coordena agregados) ou método na raiz. Use Case chamando outro acopla intenções e cria transações aninhadas; é cheiro de modelagem.

## Por que importa

O ponto operacional é o que se ganha após 2-3 anos:

1. **Legibilidade pela ementa.** Um dev novo entra no projeto, abre `src/Application/`, lê os nomes dos diretórios e Use Cases — e sabe **o que o sistema faz** sem ler código. Pasta = funcionalidade. Sem essa estrutura, o conhecimento mora em controllers gordos onde caso de uso e detalhes HTTP se misturam.

2. **Refatoração segura.** Mover uma feature de "feita no painel admin" para "agendada via worker" é trocar adapter de entrada. O Use Case não muda. Sem ele, lógica está colada ao controller e a mudança vira cópia.

3. **Auditoria e log uniformes.** Como todo fluxo de mutação passa por Handler, um *decorator* logger/auditor envolvendo todos os handlers via container resolve auditoria de uma vez. Idem para retry, métricas, autorização, idempotência. Sem ponto de entrada uniforme, cada controller implementa do seu jeito (ou esquece).

4. **Onboarding e estimativa.** "Adicionar feature X" = "criar Command + Handler + porta nova se precisar". O custo cognitivo de mudança fica previsível.

## Exemplo de código PHP

### 1. Command e Handler básicos

```php
<?php
declare(strict_types=1);

namespace App\Application\Order\Cancel;

use App\Domain\Order\OrderId;
use App\Domain\User\UserId;

// Command: VOs no lugar de primitivas, readonly
final readonly class CancelOrderCommand
{
    public function __construct(
        public OrderId $orderId,
        public string $reason,
        public UserId $cancelledBy,
        public ?string $idempotencyKey = null,
    ) {}
}
```

```php
<?php
declare(strict_types=1);

namespace App\Application\Order\Cancel;

use App\Domain\Order\{OrderNotFound, OrderRepository};
use App\Domain\Shared\Clock;
use App\Application\Shared\EventBus;

final readonly class CancelOrderHandler
{
    public function __construct(
        private OrderRepository $orders,
        private EventBus $bus,
        private Clock $clock,
    ) {}

    public function __invoke(CancelOrderCommand $cmd): void
    {
        // 1. carrega o agregado
        $order = $this->orders->ofId($cmd->orderId)
            ?? throw OrderNotFound::withId($cmd->orderId);

        // 2. delega a REGRA ao agregado (não escreva if aqui)
        $order->cancel($cmd->reason, $cmd->cancelledBy, $this->clock->now());

        // 3. persiste
        $this->orders->save($order);

        // 4. publica eventos coletados
        foreach ($order->pullEvents() as $event) {
            $this->bus->dispatch($event);
        }
    }
}
```

### 2. Query Handler (lado de leitura)

```php
<?php
declare(strict_types=1);

namespace App\Application\Order\List;

use App\Domain\User\UserId;

final readonly class ListRecentOrdersQuery
{
    public function __construct(
        public UserId $customer,
        public int $limit = 20,
    ) {}
}

// Resultado: read model puro, não entidade
final readonly class OrderListItem
{
    public function __construct(
        public string $id,
        public int $totalCents,
        public string $status,
        public \DateTimeImmutable $createdAt,
    ) {}
}

final readonly class ListRecentOrdersHandler
{
    public function __construct(private OrderQueryPort $query) {}

    /** @return list<OrderListItem> */
    public function __invoke(ListRecentOrdersQuery $q): array
    {
        // Não carrega agregado — vai direto na projeção otimizada
        return $this->query->recentByCustomer($q->customer, $q->limit);
    }
}
```

### 3. Decorator transacional (orto­gonal ao Handler)

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Application;

final readonly class TransactionalHandler
{
    public function __construct(
        private object $inner,           // o handler real
        private \PDO $pdo,
    ) {}

    public function __invoke(object $cmd): mixed
    {
        $this->pdo->beginTransaction();
        try {
            $result = ($this->inner)($cmd);
            $this->pdo->commit();
            return $result;
        } catch (\Throwable $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }
}
```

O container envolve todos os handlers de comando com esse decorator. Cada Use Case se preocupa só com a coreografia; transação é infra.

### 4. Decorator de idempotência

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Application;

final readonly class IdempotentHandler
{
    public function __construct(
        private object $inner,
        private IdempotencyStore $store,
    ) {}

    public function __invoke(object $cmd): mixed
    {
        // Convenção: comando carrega chave opcional
        $key = $cmd->idempotencyKey ?? null;

        if ($key !== null && $this->store->has($key)) {
            return $this->store->get($key);
        }

        $result = ($this->inner)($cmd);

        if ($key !== null) {
            $this->store->put($key, $result);
        }

        return $result;
    }
}
```

### 5. Anti-padrão: Use Case carregando regra

```php
<?php
declare(strict_types=1);

// RUIM — regra dentro do Use Case
final class PayOrderHandler
{
    public function __invoke(PayOrderCommand $cmd): void
    {
        $order = $this->orders->ofId($cmd->orderId);

        // CHEIRO: condicional de regra no Use Case
        if ($order->status === 'cancelled') {
            throw new \DomainException('cannot pay cancelled');
        }
        if ($order->total > 1000_00) {
            $order->requiresApproval = true;
        }

        // ...
    }
}

// BOM — Use Case só coreografa, regra mora no AR
final class PayOrderHandler
{
    public function __invoke(PayOrderCommand $cmd): void
    {
        $order = $this->orders->ofId($cmd->orderId);
        $order->pay($this->gateway, $this->clock->now());  // AR sabe sua regra
        $this->orders->save($order);
    }
}
```

## Quando usar / quando não usar

- **Use** sempre em sistemas hexagonais / DDD — é o ponto de entrada canônico do núcleo.
- **Use** quando o sistema é acessado por ≥ 2 canais (HTTP + CLI + worker). A reusabilidade é gratuita.
- **Use** quando há expectativa de auditoria/log uniforme — decorator único resolve.
- **Não use** estrutura completa em CRUD trivial. "Listar usuários" como Use Case com Command + Handler + DTO pode ser teatro. Seja consistente porém: ou todos os fluxos passam por Use Case, ou nenhum. Mistura cria zonas cinzentas onde regra escapa.
- **Não use** Use Case para script de migração ou job de manutenção único.

## Armadilhas comuns

- **Use Case com regra de negócio.** `if ($order->total > 1000) ...` dentro do handler. Regra é do agregado. Use Case orquestra.
- **Command com primitivas.** `public string $orderId` aceita qualquer string. Use `OrderId` (VO) — validação acontece no construtor do VO, falha cedo, type-system trabalha por você.
- **Retornar entidade do domínio para o controller.** Vaza modelo. Devolva DTO/Read Model. Controller serializa o DTO; nunca o agregado direto.
- **Use Case chamando Use Case.** Cheiro de modelagem. A regra compartilhada vira Domain Service. Se for só conveniência, replique código — duplicação é melhor que acoplamento errado.
- **Mistura Command/Query.** Handler que lê coisa pesada e escreve no mesmo passo. Separe. Query Handler para leitura otimizada; Command Handler para mutação.
- **Transação fora do Use Case.** `beginTransaction` no controller. Quebra reutilização (CLI/worker reimplementa). Coloque em decorator.
- **Esquecer idempotência em fluxos expostos a retry.** Webhook do gateway retentando, cliente HTTP retentando após timeout, worker reentregando mensagem — duplicação de pagamento, e-mails repetidos. Sempre considere idempotencyKey em comandos que mutam estado externo.
- **Handler gigante "WorkflowHandler".** Quando o Use Case orquestra 8 passos e tem 200 linhas, ele esconde múltiplos casos de uso. Divida.

## Links relacionados

- [[Camadas-DDD]]
- [[Hexagonal]]
- [[Aggregates]]
- [[Domain-Events-CQRS]]
- [[APIs/REST-Maduro]]
- [[Padroes/Repository]]

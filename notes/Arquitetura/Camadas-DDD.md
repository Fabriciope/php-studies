---
tags: [php, arquitetura, ddd, clean]
fase: 4
status: draft
---

# Camadas: Domain / Application / Infrastructure / Presentation

## Conceito

A ideia de separar software em **camadas concêntricas com dependência unidirecional** aparece em três tradições principais que convergem para o mesmo princípio: Eric Evans (*Domain-Driven Design*, 2003) com o "Layered Architecture pattern"; Jeffrey Palermo com a *Onion Architecture* (2008); e Robert C. Martin (*Uncle Bob*) com a *Clean Architecture* (2012). Alistair Cockburn já havia descrito a *Hexagonal Architecture* (Ports & Adapters, 2005) trazendo o foco para a direção das dependências. Apesar de vocabulário distinto, todas afirmam o mesmo: **o núcleo de regras de negócio não deve depender de detalhes técnicos** (banco, framework, protocolo).

O modelo mental é o de **círculos concêntricos**. Quanto mais para dentro, mais estável e mais próximo das regras de negócio; quanto mais para fora, mais volátil e mais próximo da tecnologia. Setas de dependência **só atravessam de fora para dentro**. Quando o núcleo precisa de algo do exterior (ex.: persistir, enviar e-mail), ele **declara uma interface** (porta) e a infraestrutura **implementa**. Isso é a *Dependency Inversion Principle* operando em nível arquitetural.

```
              ┌──────────────────────────────────────┐
              │           Presentation               │
              │  (HTTP Controller, CLI, Worker)      │
              │     ┌──────────────────────────┐     │
              │     │      Application         │     │
              │     │   (Use Cases / Handlers) │     │
              │     │   ┌──────────────────┐   │     │
              │     │   │     Domain       │   │     │
              │     │   │  AR, VO, Events  │   │     │
              │     │   │  Portas (iface)  │   │     │
              │     │   └──────────────────┘   │     │
              │     └──────────────────────────┘     │
              │           Infrastructure             │
              │  (PDO, HTTP client, Mailer, Queue)   │
              └──────────────────────────────────────┘
                  dependência aponta para dentro →
```

As quatro camadas com responsabilidades distintas:

- **Domain** — coração. Entidades, agregados, VOs, eventos de domínio, *domain services* e **interfaces** (portas). Zero framework, zero I/O, zero `Request`/`Response`. Se rodar em CLI, web, fila ou Lambda — o Domain não percebe.
- **Application** — orquestra **um caso de uso** por vez. Recebe um Command (DTO), carrega agregados pelo repositório, chama métodos do domínio, persiste e despacha eventos. Sem regra de negócio, sem framework. É a "fina camada de coreografia".
- **Infrastructure** — implementa as portas declaradas no Domain (`PdoOrderRepository implements OrderRepository`). Aqui mora SQL, cliente Stripe, SDK da AWS, mailer SMTP, conexão Redis.
- **Presentation** — adapta o mundo externo para a Application. Controller HTTP traduz `Request` em Command, chama o Use Case, traduz Result em `Response`. CLI faz o mesmo a partir de `argv`. Worker faz o mesmo a partir de mensagem de fila.

DDD ainda separa o pensamento em duas dimensões. **DDD Estratégico** trata do macro: *Ubiquitous Language* (a equipe fala a linguagem do negócio sem tradução), *Bounded Contexts* (cada subdomínio com seu modelo próprio, sem modelo único global), *Context Map* (como contextos se relacionam: shared kernel, customer-supplier, anti-corruption layer, conformist). **DDD Tático** é o que esta nota mostra: agregados, VOs, eventos, repositórios. Time que pula o estratégico e adota só o tático costuma produzir CRUD com ornamentação — caro e sem benefício.

Existe também a relação com a *Conway's Law* ("organizações desenham sistemas que copiam suas estruturas de comunicação"). Bounded Contexts costumam mapear para times. Se um time mexe em dois contextos toda sprint, ou a fronteira está errada ou o time está mal-formado.

A *Anti-Corruption Layer* (ACL) merece destaque: quando seu contexto consome um sistema legado/externo cuja linguagem é diferente, **não importe o modelo deles**. Crie um adapter que traduz para o seu vocabulário. Sem ACL, o vocabulário externo contamina o domínio em meses.

## Por que importa

O argumento prático é de **manutenção em 2-3 anos**. Imagine três cenários do mesmo sistema vivo por 36 meses:

1. **Troca de framework.** Symfony 7 vai virar Symfony 8; ou o time decide migrar para Laravel; ou um serviço crítico precisa virar um worker standalone. Se `Order` extende `Doctrine\ORM\EntityRepository` e o controller injeta `EntityManagerInterface`, a migração custa semanas. Se o Domain só importa `App\Domain\*`, troca-se apenas Infrastructure e Presentation — questão de dias.

2. **Mudança de banco.** PostgreSQL para Aurora; tabela para MongoDB para um agregado específico; cache de leitura via Redis. Se SQL está no Use Case (`$pdo->query(...)`), cada query vira refactor. Se está atrás de `OrderRepository`, troca-se a implementação.

3. **Reuso entre canais.** O Use Case `CancelOrder` originalmente foi escrito para um endpoint HTTP. Agora o suporte precisa cancelar via CLI; o webhook do gateway de pagamento também aciona cancelamento; uma rotina diária cancela pedidos abandonados. Com camadas, cada canal é um adapter de entrada que constrói o Command — zero duplicação. Sem camadas, copia-se a lógica do controller três vezes e elas divergem.

Em termos de **testabilidade**, o Domain testado sem subir banco, sem mockar framework, sem fixtures HTTP: testes de 5ms que rodam aos milhares. A pirâmide de testes começa a fazer sentido. Sem separação, todo teste é integração — lento, frágil, com mocks profundos.

Há um custo real: cerimônia. Adicionar um campo num CRUD trivial passa de "alterar o controller" para "alterar Command + Use Case + AR + Repository + Migration". Esse custo paga em sistemas com **regra de negócio rica**; em CRUD puro vira teatro.

## Exemplo de código PHP

Estrutura de diretórios canônica (Symfony-style, mas independe de framework):

```
src/
├── Domain/                              ← núcleo, sem framework
│   ├── Order/
│   │   ├── Order.php                    ← Aggregate Root
│   │   ├── OrderId.php                  ← Value Object
│   │   ├── OrderStatus.php              ← Enum
│   │   ├── OrderItem.php                ← entity dentro do AR
│   │   ├── OrderRepository.php          ← interface (porta secundária)
│   │   ├── Exception/OrderNotFound.php
│   │   └── Events/OrderPaid.php         ← Domain Event
│   ├── Payment/
│   │   └── PaymentGateway.php           ← porta secundária
│   └── Shared/
│       ├── Money.php                    ← VO compartilhado
│       └── Clock.php                    ← porta para tempo (testabilidade)
│
├── Application/                         ← orquestração, sem framework
│   ├── Order/
│   │   ├── Pay/
│   │   │   ├── PayOrderCommand.php      ← DTO de entrada
│   │   │   ├── PayOrderHandler.php      ← Use Case
│   │   │   └── PayOrderResult.php       ← DTO de saída (opcional)
│   │   └── List/
│   │       ├── ListOrdersQuery.php
│   │       └── ListOrdersHandler.php
│   └── Shared/
│       └── EventBus.php                 ← porta de aplicação
│
├── Infrastructure/                      ← adapters concretos
│   ├── Persistence/
│   │   ├── Doctrine/DoctrineOrderRepository.php
│   │   └── InMemory/InMemoryOrderRepository.php  ← útil para testes
│   ├── Payment/StripePaymentGateway.php
│   ├── Bus/SymfonyEventBus.php
│   └── Clock/SystemClock.php
│
└── Presentation/                        ← entrada do mundo
    ├── Http/
    │   ├── PayOrderController.php
    │   └── ListOrdersController.php
    ├── Cli/
    │   └── RetryFailedPaymentsCommand.php
    └── Worker/
        └── ProcessPaymentWebhookHandler.php
```

A regra de dependência, materializada num teste com Deptrac/PHPStan ou puramente convencional:

```php
<?php
declare(strict_types=1);

// Domain NÃO importa Application, Infrastructure ou Presentation
// Application importa APENAS Domain
// Infrastructure importa Domain e Application (para implementar portas)
// Presentation importa Application (e VOs de Domain para input)
```

Exemplo de porta declarada no Domain e implementada na Infrastructure:

```php
<?php
declare(strict_types=1);

namespace App\Domain\Order;

// Porta: contrato do que o domínio precisa do mundo
interface OrderRepository
{
    public function ofId(OrderId $id): ?Order;

    public function save(Order $order): void;

    /** @return list<Order> */
    public function pendingOlderThan(\DateTimeImmutable $cutoff): array;
}
```

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Persistence\Doctrine;

use App\Domain\Order\{Order, OrderId, OrderRepository};
use Doctrine\ORM\EntityManagerInterface;

// Adapter: traduz entre Domain e detalhe técnico
final readonly class DoctrineOrderRepository implements OrderRepository
{
    public function __construct(private EntityManagerInterface $em) {}

    public function ofId(OrderId $id): ?Order
    {
        return $this->em->find(Order::class, $id->value);
    }

    public function save(Order $order): void
    {
        $this->em->persist($order);
        $this->em->flush();
    }

    public function pendingOlderThan(\DateTimeImmutable $cutoff): array
    {
        return $this->em->createQuery(
            'SELECT o FROM ' . Order::class . ' o
             WHERE o.status = :pending AND o.createdAt < :cutoff'
        )
            ->setParameters(['pending' => 'pending', 'cutoff' => $cutoff])
            ->getResult();
    }
}
```

Um adapter de entrada (Controller) traduzindo HTTP em Command:

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
        // Presentation cuida de HTTP: parse, validação de forma, status code
        $body = json_decode((string) $req->getBody(), true, flags: JSON_THROW_ON_ERROR);

        // Traduz para vocabulário do domínio
        $cmd = new PayOrderCommand(
            orderId: new OrderId($body['order_id']),
            paymentMethod: $body['method'],
        );

        $this->handler->__invoke($cmd);

        // HTTP-specific: status code
        return new JsonResponse(['status' => 'accepted'], 202);
    }
}
```

## Comparação entre abordagens

| Aspecto              | DDD Layered     | Clean Architecture | Onion          | Hexagonal      |
|----------------------|-----------------|--------------------|----------------|----------------|
| Origem               | Evans 2003      | Uncle Bob 2012     | Palermo 2008   | Cockburn 2005  |
| Núcleo               | Domain          | Entities + UseCase | Domain Model   | Application    |
| Foco principal       | Modelo do negócio | Regra de dep.    | Inversão       | Direção de I/O |
| Linguagem            | Camadas         | Círculos           | Anéis          | Portas/Adapt.  |
| Vocabulário "porta"  | ausente formal  | Boundary           | ausente        | Port           |
| Convergência prática | Todas dizem o mesmo: dependência aponta para o núcleo, infra implementa |

Na prática, projetos modernos misturam: **DDD tático** para modelar (AR, VO, eventos), **Hexagonal** para descrever direção de dependências (portas), **Clean** para o "use case = unidade de trabalho", **Onion** apenas como mnemônico visual.

## Quando usar / quando não usar

- **Use** em sistemas com lógica de negócio não trivial e ciclo de vida > 1 ano.
- **Use** quando há ≥ 2 canais de entrada (HTTP + CLI + worker) ou ≥ 2 adapters de saída (banco + cache + API externa) — a separação se paga sozinha.
- **Use** quando o time tem ≥ 3 devs ativos no mesmo código — a estrutura organiza colaboração.
- **Não use** estrutura completa em CRUD puro (admin de cadastros, formulário → tabela). Custo de cerimônia maior que retorno.
- **Não use** em scripts utilitários, jobs únicos, MVPs descartáveis.
- **Microsserviço pequeno** pode colapsar Application em Domain ou ter apenas dois pacotes (`Core`, `Adapters`). Não é dogma.

## Armadilhas comuns

- **ORM-Entity vazando para Domain.** Anotações Doctrine (`#[ORM\Entity]`, `#[ORM\Column]`) no agregado prendem o Domain à versão do Doctrine. Use mapeamento por XML/YAML, ou converse com a equipe sobre o trade-off (anotações são pragmáticas mas custam pureza).
- **"Service Layer" como saco de utilitários.** Classes `OrderService` com 20 métodos pouco coesos não são Application Service. Application = um Use Case por classe.
- **`Request`/`Response` no Domain.** Importar `Symfony\Component\HttpFoundation\Request` em qualquer classe de Domain é game over. O Domain não sabe que existe HTTP.
- **Repository devolvendo array associativo.** Quebra o contrato do agregado. Repository devolve agregado (ou `null`), nada de "DTOs cruas".
- **Use Case chamando Use Case.** Cheiro de regra de negócio mal modelada. A regra compartilhada vira *Domain Service* ou método no agregado.
- **Validação duplicada.** Se a regra "qty > 0" mora no controller, no Use Case e no AR, a sincronia se perde. **Invariante mora no agregado**; o resto valida forma (formato de string, presença de campo).
- **Não enforce do limite de camada.** Sem Deptrac/PHPStan/lint, em 6 meses alguém importa `Domain` em `Presentation` ou vice-versa. Automatize.
- **Bounded Contexts esquecidos.** Tudo num único módulo gigante. Quando o sistema cresce, separar contextos depois custa caro. Pense em fronteiras desde o início.

## Links relacionados

- [[Hexagonal]]
- [[Use-Cases]]
- [[Aggregates]]
- [[Value-Objects]]
- [[Domain-Events-CQRS]]
- [[Padroes/Repository]]

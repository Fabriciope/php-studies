---
tags: [php, arquitetura, ddd, clean]
fase: 4
status: stub
---

# Camadas: Domain / Application / Infrastructure / Presentation

## Conceito

Quatro camadas com dependências unidirecionais (sempre apontando para dentro):

- **Domain** — regras de negócio puras. Sem framework, sem I/O. Entidades, VOs, agregados, eventos, interfaces (portas).
- **Application** — orquestra casos de uso. Recebe comandos, coordena domínio e portas. Sem framework também.
- **Infrastructure** — adapters concretos das portas (PDO, HTTP client, fila, mailer).
- **Presentation** — entrada do mundo externo: controllers HTTP, CLI, workers de fila.

## Por que importa

A camada errada pegando regra errada é o que transforma sistemas em pântanos: HTTP concern vazando pra domínio, SQL no controller, framework dentro do agregado. As camadas dão **um lugar para cada coisa** e tornam o "vai mudar de framework" um non-event.

## Exemplo de código PHP

```
src/
├── Domain/
│   ├── Order/
│   │   ├── Order.php              ← agregado
│   │   ├── OrderId.php            ← VO
│   │   ├── OrderStatus.php        ← Enum
│   │   ├── OrderRepository.php    ← interface (porta)
│   │   └── Events/OrderPaid.php
│   └── Shared/
│       └── Money.php
├── Application/
│   └── Order/
│       ├── PayOrder.php           ← use case (handler)
│       └── PayOrderCommand.php
├── Infrastructure/
│   ├── Persistence/PdoOrderRepository.php
│   ├── Payment/StripePaymentGateway.php
│   └── Bus/SymfonyEventBus.php
└── Presentation/
    ├── Http/PayOrderController.php
    └── Cli/RetryFailedPaymentsCommand.php
```

```php
// regra: Domain NÃO importa Application/Infra/Presentation
// regra: Application importa Domain (apenas)
// regra: Infrastructure implementa portas de Domain
// regra: Presentation chama Application
```

## Quando usar / quando não usar

- **Sempre** em sistema com regra de negócio não trivial e ciclo de vida > 1 ano.
- **Não usar** essa estrutura completa em CRUD trivial — gera mais cerimônia que valor.
- **Microsserviço pequeno** pode colapsar Application em Domain (Use Case mora junto da entidade) — não é dogma.

## Armadilhas comuns

- ORM-Entity vazando pra domínio: anotações de Doctrine/atributos de mapeamento contaminam VO. Use mapeadores explícitos quando der.
- "Service Layer" que vira saco de utilitários — não é Application Service. Se a classe não orquestra um caso de uso, não pertence à Application.
- Importar `Symfony\Component\HttpFoundation\Request` no Domain — fim de jogo.

## Links relacionados

- [[Hexagonal]]
- [[Use-Cases]]
- [[Aggregates]]

---
tags: [php, padroes, solid]
fase: 3
status: stub
---

# SOLID

## Conceito

Cinco princípios de design OO:

- **S**ingle Responsibility — uma classe, uma razão para mudar
- **O**pen/Closed — aberto para extensão, fechado para modificação
- **L**iskov Substitution — subtipo intercambiável pelo supertipo
- **I**nterface Segregation — clientes não dependem de métodos que não usam
- **D**ependency Inversion — dependa de abstrações, não de concretas

## Por que importa

SOLID é vocabulário compartilhado. Não são leis — são heurísticas para detectar **acoplamento perigoso**. Aplicados sem critério, viram over-engineering. Aplicados com tato, fazem o sistema absorver mudança sem rachar.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// ❌ violação: SRP + DIP — calcula E persiste E envia email
final class Order {
    public function checkout(): void {
        $total = /* ... cálculo ... */;
        (new MySQLConnection())->save($this);
        mail($this->customerEmail, 'Pedido', '...');
    }
}

// ✅ SRP: cada classe tem UMA razão de mudar
// ✅ DIP: depende de interfaces (abstração), não de MySQL
interface OrderRepository { public function save(Order $order): void; }
interface Mailer { public function send(string $to, string $subject, string $body): void; }

final readonly class CheckoutOrder {
    public function __construct(
        private OrderRepository $orders,
        private Mailer $mailer,
    ) {}

    public function __invoke(Order $order): void {
        $this->orders->save($order);
        $this->mailer->send($order->customerEmail, 'Pedido', '...');
    }
}
```

## Quando usar / quando não usar

- **SRP** — sempre, é o filtro mais valioso.
- **OCP** — quando o ponto de variação é claro (estratégias de desconto). Não invente extensão para o que nunca varia.
- **LSP** — silencioso e crítico: subclasse que rejeita argumento aceito pelo pai é violação.
- **ISP** — `interface Repository` com 12 métodos vira lixo. Quebre por uso real.
- **DIP** — invista no boundary com mundo externo (banco, HTTP, FS). Domínio puro **não** precisa de interface para tudo.

## Armadilhas comuns

- "Classe SRP" virar 200 classes de 5 linhas — você não dividiu, atomizou.
- Criar interface só porque "DI". Se há **uma** implementação real e nunca haverá outra, é overhead.
- LSP não é só assinatura — é **contrato**. Subclasse que torna pré-condição mais forte ou pós-condição mais fraca quebra.

## Links relacionados

- [[Strategy]]
- [[Repository]]
- [[Arquitetura/Hexagonal]]

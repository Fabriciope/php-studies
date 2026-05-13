---
tags: [php, padroes, solid]
fase: 3
status: draft
---

# SOLID

## Conceito

SOLID é um acrônimo cunhado por Robert C. Martin no início dos anos 2000, agregando cinco princípios de design orientado a objetos. Cada letra resume uma heurística para reduzir acoplamento, aumentar coesão e absorver mudança sem fratura no sistema. Não são leis físicas: são lentes para diagnosticar quando um pedaço de código está prestes a se tornar caro de manter.

- **S**ingle Responsibility — uma classe deve ter uma única razão para mudar.
- **O**pen/Closed — entidades de software devem ser abertas para extensão e fechadas para modificação.
- **L**iskov Substitution — um subtipo deve ser intercambiável pelo supertipo sem quebrar contratos.
- **I**nterface Segregation — clientes não devem depender de métodos que não usam.
- **D**ependency Inversion — módulos de alto nível e de baixo nível devem depender de abstrações; abstrações não devem depender de detalhes.

O modelo mental útil é pensar em **eixos de mudança**. SRP pergunta "quem pede mudança nesta classe?". OCP pergunta "para extender comportamento, eu modifico ou compor?". LSP pergunta "este filho honra o contrato do pai?". ISP pergunta "esta interface obriga clientes a conhecer o que não precisam?". DIP pergunta "quem aponta para quem na minha dependência: o domínio puxa o detalhe ou o detalhe puxa o domínio?".

```
+--------------------+         +--------------------+
|   Domínio (alto)   | -----> | Abstração (porta) |
+--------------------+         +--------------------+
                                       ^
                                       | implementa
                                       |
                              +--------------------+
                              | Adapter (PDO/HTTP) |
                              +--------------------+
```

Críticos modernos como **Dan North** ("CUPID") argumentam que SOLID superestima OO clássico e subestima composição e funções. A objeção é justa quando SOLID vira ritual: criar interface para cada classe, fragmentar serviços até pulverizar coesão. Como toda heurística, SOLID rende dividendos quando aplicado com contexto — em código de domínio, na borda do sistema, em pontos de variação real — e desperdiça quando aplicado em scripts triviais ou CRUD sem regra. A gradação importa: nem tudo merece a cerimônia.

Comparado a princípios irmãos: **DRY** mira duplicação, **YAGNI** mira especulação, **KISS** mira complexidade acidental. SOLID se sobrepõe a todos, mas seu foco é **forma da dependência**.

## Por que importa

Quando uma classe acumula motivos para mudar (SRP), uma alteração de regra fiscal exige tocar onde se faz envio de email. Quando OCP é violado, adicionar uma forma de pagamento exige editar um `switch` central de 300 linhas — e quebrar testes não relacionados. LSP violado aparece como `if ($x instanceof Subtipo)` espalhado: prova de que o polimorfismo prometido não funciona. ISP violado força mocks gigantes em testes (`MockRepository` com 12 métodos quando o teste usa um). DIP violado faz o domínio carregar `mysqli_*` no import, impedindo teste offline.

Cenários concretos:

- Em **uma API de e-commerce**, sem DIP, trocar Stripe por Pagar.me vira projeto de mês porque o use case importa o SDK direto. Com DIP, o use case fala com `PaymentGateway` (interface) e o adapter muda.
- Em **um sistema legado**, SRP mal aplicado faz o `class User` ter 80 métodos (auth, perfil, billing, gamificação). Refatorar começa por separar responsabilidades — não atomizando em 80 classes, mas em ~5 colaboradores coesos.
- Em **um pacote reutilizável**, ISP mal feita força quem só lê dados a implementar métodos de escrita. Quebrar em `OrderReader` e `OrderWriter` libera consumidores read-only.

## Exemplo de código PHP

### 1) SRP: separar razões de mudança

```php
<?php
declare(strict_types=1);

// Anti-pattern: três razões de mudar na mesma classe.
// Mudou cálculo? Mudou persistência? Mudou notificação? Tudo aqui.
final class OrderBad
{
    public function checkout(): void
    {
        $total = /* cálculo complexo */ 0;
        (new \mysqli('host', 'u', 'p', 'db'))->query("INSERT ...");
        mail($this->customerEmail, 'Pedido', '...');
    }
}

// Refatorado: cada colaborador tem uma razão.
interface OrderRepository
{
    public function save(Order $order): void;
}

interface Mailer
{
    public function send(string $to, string $subject, string $body): void;
}

final readonly class CheckoutOrder
{
    public function __construct(
        private OrderRepository $orders,
        private Mailer $mailer,
    ) {}

    public function __invoke(Order $order): void
    {
        $this->orders->save($order);
        $this->mailer->send($order->customerEmail, 'Pedido confirmado', '...');
    }
}
```

### 2) OCP: estender por composição, não modificação

```php
<?php
declare(strict_types=1);

// Anti-pattern: cada novo tipo de desconto exige editar o método.
final class PricingBad
{
    public function total(Cart $cart, string $kind): int
    {
        $sub = $cart->subtotalCents();
        return match ($kind) {
            'none'    => $sub,
            'percent' => (int) ($sub * 0.9),
            'fixed'   => max(0, $sub - 1000),
            // amanhã: 'bogo', 'tier', 'voucher' ...
        };
    }
}

// OCP: fecha-se a classe, abre-se via interface.
interface Discount
{
    public function applyCents(int $subtotal): int;
}

final readonly class PercentOff implements Discount
{
    public function __construct(private int $percent) {}
    public function applyCents(int $sub): int
    {
        return (int) ($sub * (100 - $this->percent) / 100);
    }
}

final readonly class Pricing
{
    public function total(Cart $cart, Discount $discount): int
    {
        return $discount->applyCents($cart->subtotalCents());
    }
}
```

### 3) LSP: contravariância de parâmetro e covariância de retorno

```php
<?php
declare(strict_types=1);

// PHP 7.4+ permite covariância de retorno e contravariância de parâmetro
// em sobrescritas. Isso é a forma sintática de LSP.

class Animal {}
class Cachorro extends Animal {}

interface Abrigo
{
    public function aceita(Cachorro $a): Animal;
}

// Subtipo aceita argumento MAIS GERAL (contravariância) — ok pelo LSP,
// e retorna tipo MAIS ESPECÍFICO (covariância) — também ok.
final class AbrigoGeneroso implements Abrigo
{
    public function aceita(Animal $a): Cachorro
    {
        return new Cachorro();
    }
}

// Anti-pattern LSP: subclasse fortalece pré-condição, viola contrato.
class Repo
{
    public function save(Order $o): void { /* aceita qualquer Order */ }
}

class RepoStrict extends Repo
{
    public function save(Order $o): void
    {
        // Rejeita o que o pai aceitaria: violação semântica de LSP.
        if ($o->totalCents() <= 0) {
            throw new \DomainException('total must be positive');
        }
        parent::save($o);
    }
}
```

### 4) ISP: quebrar interfaces gordas por uso real

```php
<?php
declare(strict_types=1);

// Anti-pattern: clientes read-only carregam métodos de escrita inúteis.
interface OrderRepositoryFat
{
    public function save(Order $o): void;
    public function delete(OrderId $id): void;
    public function ofId(OrderId $id): ?Order;
    public function listAll(): iterable;
    public function archive(OrderId $id): void;
    public function statistics(): array;
}

// ISP: segregar por papel.
interface OrderReader
{
    public function ofId(OrderId $id): ?Order;
    /** @return iterable<Order> */
    public function listAll(): iterable;
}

interface OrderWriter
{
    public function save(Order $o): void;
    public function delete(OrderId $id): void;
}

// Implementação única pode realizar as duas:
final class PdoOrderRepository implements OrderReader, OrderWriter
{
    // ...
}

// Mas o relatório (read-only) só depende do que precisa:
final readonly class OrderReport
{
    public function __construct(private OrderReader $orders) {}
}
```

### 5) DIP: domínio aponta para abstração

```php
<?php
declare(strict_types=1);

// Anti-pattern: use case importa adapter concreto.
final class ConfirmPaymentBad
{
    public function run(PaymentRequest $r): void
    {
        $stripe = new \Stripe\StripeClient('sk_...');  // domínio amarrado ao SDK
        $stripe->charges->create([/* ... */]);
    }
}

// DIP: defina a porta no domínio, adapte fora.
interface PaymentGateway
{
    public function charge(Money $amount, string $token): PaymentReceipt;
}

final readonly class ConfirmPayment
{
    public function __construct(private PaymentGateway $gateway) {}

    public function __invoke(PaymentRequest $r): PaymentReceipt
    {
        return $this->gateway->charge($r->amount, $r->token);
    }
}

// Adapter mora na borda (Infrastructure), não no domínio.
final readonly class StripeGateway implements PaymentGateway
{
    public function __construct(private \Stripe\StripeClient $client) {}
    public function charge(Money $a, string $t): PaymentReceipt { /* ... */ }
}
```

## Quando usar / quando não usar

- **SRP**: aplicar sempre que detectar dois grupos distintos pedindo mudança na mesma classe. Não confundir com "uma classe = um método".
- **OCP**: aplicar onde a variação é real e recorrente (estratégias de desconto, formas de pagamento, formatos de exportação). Não inventar pontos de extensão para variação imaginária — viola YAGNI.
- **LSP**: aplicar sempre, é silencioso e estrutural. Toda hierarquia ou implementação de interface precisa honrar o contrato (pré-condições, pós-condições, invariantes).
- **ISP**: aplicar quando consumidores reais não usam metade da interface, ou quando mocks ficam gigantes. Especialmente crítico em interfaces de Repository (veja [[Repository]]).
- **DIP**: aplicar no boundary com o mundo externo (banco, HTTP, FS, fila, cache, clock, random). Para colaboração interna ao domínio, criar interface só porque é "boa prática" gera ruído sem ganho.

Gradação: em prototipagem ou script ETL one-off, SOLID é desperdício. Em código que vai viver anos, é seguro de mudança.

## Armadilhas comuns

- **Atomização disfarçada de SRP**: dividir uma classe em 200 classes de 5 linhas não é SRP, é dispersão. SRP é sobre razão de mudança, não tamanho.
- **Interface especulativa**: criar `FooInterface` com uma única implementação real e nenhuma perspectiva de segunda. Custo de cerimônia sem retorno; adicione quando a segunda nascer.
- **LSP só sintático**: assinatura compatível não basta. Subclasse que rejeita entradas aceitas pelo pai, ou que torna pós-condição mais fraca, viola LSP mesmo passando no parser.
- **ISP confundido com SRP**: ISP é sobre **interfaces**, SRP é sobre **classes**. Uma classe pode implementar várias interfaces segregadas mantendo coesão.
- **DIP virado de cabeça**: "inverter" dependência colocando a interface no pacote do adapter, não do domínio. A interface pertence a quem **consome**, não a quem implementa.
- **SOLID como ritual**: aplicar os cinco em código que não precisa, gerando indireção sem ganho. Cada princípio deve ser justificado por uma dor real.
- **Construtor com 12 dependências**: sintoma de SRP violado na classe que injeta tudo. Quebre antes de injetar mais.
- **Service Locator escondido como DIP**: classe recebendo `ContainerInterface` no construtor não é DIP, é Service Locator — anti-pattern. Injete o que precisa, não o container.

## Links relacionados

- [[Strategy]]
- [[Repository]]
- [[Factory]]
- [[Decorator]]
- [[Specification]]
- [[Arquitetura/Hexagonal]]
- [[Arquitetura/Use-Cases]]

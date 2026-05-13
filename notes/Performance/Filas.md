---
tags: [php, performance, filas, async]
fase: 6
status: draft
---

# Filas (Redis / RabbitMQ)

## Conceito

Fila é a materialização do padrão **Producer/Consumer** (produtor/consumidor) — um buffer entre quem **produz trabalho** (geralmente uma rota HTTP, evento de domínio, scheduler) e quem **executa trabalho** (worker em loop, frequentemente em outro processo ou servidor). A finalidade é desacoplar tempo de resposta (resposta HTTP rápida) da execução real (processamento que pode levar segundos a minutos), absorver picos (produtor pode produzir mais rápido do que consumidor processa, sem perda), e dar resiliência (retry, dead-letter, idempotência) a operações flaky (e-mails SMTP, webhooks, APIs externas instáveis).

O elemento central é o **broker**, o sistema que armazena e entrega mensagens. Há quatro famílias principais no ecossistema:

| Broker      | Modelo                  | Garantia       | Throughput | Casos típicos                              |
| ----------- | ----------------------- | -------------- | ---------- | ------------------------------------------ |
| Redis Lists | Push/Pop simples        | at-most-once*  | altíssimo  | jobs simples, baixa criticidade            |
| Redis Streams | Log particionado      | at-least-once | alto       | event sourcing leve, replay                |
| RabbitMQ    | Exchange/Queue, AMQP    | at-least-once  | alto       | routing complexo, prioridade, DLX nativo   |
| Kafka       | Log distribuído         | at-least-once / exactly-once | extremo | event streaming, integração entre sistemas |
| SQS (AWS)   | Fila gerenciada         | at-least-once  | alto       | serverless, sem manutenção                 |
| Beanstalkd  | Fila simples persistente | at-least-once | médio     | legacy, simples e leve                     |

*Redis Lists com `RPOPLPUSH` para fila de processamento alcança at-least-once.

**Garantias de entrega** são o eixo central de design:

- **At-most-once**: mensagem entregue zero ou uma vez. Pode perder. Aceitável para métricas, logs onde perda parcial é ok.
- **At-least-once**: mensagem entregue uma ou mais vezes. Nunca perde, mas pode duplicar. Exige **idempotência** do consumer. Padrão da maioria dos brokers.
- **Exactly-once**: mensagem entregue exatamente uma vez. Teoricamente impossível em sistemas distribuídos (FLP impossibility); na prática, Kafka implementa "exactly-once semantics" via transações + idempotent producers, mas com restrições.

**Idempotência do consumer** é a contramedida obrigatória a at-least-once: processar a mesma mensagem N vezes produz o mesmo efeito que processar 1 vez. Implementa-se com: chaves de deduplicação (tabela `processed_messages(message_id, processed_at)`), operações naturalmente idempotentes (`UPDATE users SET email_verified=true WHERE id=42`), ou versionamento otimista.

**Visibility timeout** (SQS, RabbitMQ com ack manual): quando worker pega uma mensagem, ela some da fila por X segundos. Se worker confirma (`ack`) antes do timeout, mensagem é removida; se trava ou crasha, mensagem reaparece para outro worker. Timeout muito curto → reentrega prematura; muito longo → mensagem trava por horas se worker morrer.

**Dead Letter Queue (DLQ)** é onde mensagens "venenosas" vão depois de N tentativas falhas — fila separada para investigação humana, evitando loop infinito. **Retry com exponential backoff** é o padrão: delay = `base * 2^attempt + jitter`. Sem jitter, todas as mensagens falhas reentregam ao mesmo tempo (thundering herd).

**Outbox pattern** resolve um problema sutil: como garantir que "salvar pedido + enfileirar evento" sejam atômicos? Não dá — banco e broker são sistemas distintos. Solução: na mesma transação SQL, grave o pedido E uma linha em `outbox(id, payload, dispatched_at)`. Um worker separado lê outbox e publica no broker. Se broker estiver fora, retry. Se app crasha entre commit e publish, worker publica depois. Garante que evento *sempre* é publicado se a transação comitou (e nunca, se rolledback).

**Frameworks**: Laravel tem `Queue` (Redis, SQS, database, Beanstalkd) com workers `php artisan queue:work`. Symfony tem **Messenger** — abstração mais rica com transports plugáveis (AMQP, Redis, Doctrine, in-memory), middlewares, e suporte nativo a outbox via `doctrine_transport`. Para apps fora de framework, **php-enqueue** e **Bunny** são opções maduras.

## Por que importa

Cenário: checkout de e-commerce que envia 3 emails (confirmação, fatura, marketing), notifica WMS, atualiza ERP, dispara webhook para parceiro. Síncrono: 7s, p95 de 9s. Falha em qualquer integração = checkout falha.

Com filas:

| Métrica           | Síncrono | Async com filas |
| ----------------- | -------- | --------------- |
| p50 HTTP          | 6.2s     | 180ms           |
| p95 HTTP          | 9.4s     | 240ms           |
| Taxa de falha     | 4.8%     | 0.3%            |
| Acoplamento ao parceiro | total | nenhum   |

Quando o parceiro fica fora 30 minutos, com fila: 18 mil mensagens acumulam, parceiro volta, drena em 4 minutos. Sem fila: 18 mil checkouts falham.

Também escala separadamente: HTTP pode ter 4 servidores, workers podem ter 20 — sob carga assimétrica (muito trabalho em background, pouco tráfego HTTP), basta crescer workers.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// === Producer/Consumer com Redis Lists + retry ===

interface JobQueue
{
    public function push(string $queue, object $job, int $delay = 0): void;
}

final readonly class RedisJobQueue implements JobQueue
{
    public function __construct(private \Redis $redis) {}

    public function push(string $queue, object $job, int $delay = 0): void
    {
        $payload = serialize([
            'id'        => bin2hex(random_bytes(16)),
            'class'     => $job::class,
            'data'      => $job,
            'attempts'  => 0,
            'enqueued'  => time(),
        ]);

        if ($delay > 0) {
            // Sorted set para delayed jobs; um daemon move para fila quando ready
            $this->redis->zAdd("queue:{$queue}:delayed", time() + $delay, $payload);
        } else {
            $this->redis->rPush("queue:{$queue}", $payload);
        }
    }
}
```

```php
<?php
declare(strict_types=1);

// === Worker com retry exponencial + jitter + DLQ ===
final class Worker
{
    public function __construct(
        private \Redis $redis,
        private LoggerInterface $log,
        private JobHandlerResolver $resolver,
        private int $maxAttempts = 5,
        private int $maxRuntimeSec = 3600,    // recicla worker depois
        private int $maxJobs = 1000,          // recicla por número de jobs
    ) {}

    public function run(string $queue): void
    {
        $started = time();
        $processed = 0;

        while (
            time() - $started < $this->maxRuntimeSec
            && $processed < $this->maxJobs
            && !$this->shouldShutdown()
        ) {
            // BRPOPLPUSH = atomic pop + push para "processing list" (at-least-once)
            $raw = $this->redis->brpoplpush("queue:{$queue}", "queue:{$queue}:processing", 5);
            if ($raw === false || $raw === null) {
                continue;
            }

            $msg = unserialize($raw);
            $this->processOne($queue, $raw, $msg);
            $processed++;
        }

        $this->log->info('worker.cycle.done', ['processed' => $processed]);
    }

    private function processOne(string $queue, string $raw, array $msg): void
    {
        $context = ['job_id' => $msg['id'], 'class' => $msg['class'], 'attempt' => $msg['attempts'] + 1];

        try {
            $handler = $this->resolver->resolve($msg['class']);
            $handler->handle($msg['data']);

            // Sucesso: remover da processing list
            $this->redis->lRem("queue:{$queue}:processing", $raw, 1);
            $this->log->info('job.success', $context);
        } catch (\Throwable $e) {
            $this->redis->lRem("queue:{$queue}:processing", $raw, 1);
            $msg['attempts']++;
            $msg['last_error'] = $e->getMessage();

            if ($msg['attempts'] >= $this->maxAttempts) {
                $this->redis->rPush("queue:{$queue}:dead", serialize($msg));
                $this->log->error('job.dead_letter', $context + ['err' => $e->getMessage()]);
                return;
            }

            // Backoff exponencial com jitter (full jitter strategy)
            $base = 5; // segundos
            $cap = 3600;
            $expBackoff = min($cap, $base * (2 ** $msg['attempts']));
            $delay = random_int(0, $expBackoff);

            $this->redis->zAdd(
                "queue:{$queue}:delayed",
                time() + $delay,
                serialize($msg),
            );
            $this->log->warning('job.retry_scheduled', $context + ['delay_s' => $delay]);
        }
    }

    private function shouldShutdown(): bool
    {
        // SIGTERM/SIGINT setam flag global via pcntl_signal
        pcntl_signal_dispatch();
        return self::$shutdown ?? false;
    }

    private static bool $shutdown = false;
}
```

```php
<?php
declare(strict_types=1);

// === Idempotência do consumer ===
final readonly class IdempotentEmailSender
{
    public function __construct(
        private PDO $pdo,
        private SmtpClient $smtp,
    ) {}

    public function handle(SendOrderEmailJob $job): void
    {
        // dedup key: combinação que identifica a operação unicamente
        $dedupKey = "email.{$job->orderId}.{$job->emailType}";

        $stmt = $this->pdo->prepare(
            'INSERT IGNORE INTO processed_messages (dedup_key, processed_at)
             VALUES (:key, NOW())',
        );
        $stmt->execute(['key' => $dedupKey]);

        if ($stmt->rowCount() === 0) {
            // já processado antes: pular silenciosamente
            return;
        }

        $this->smtp->send($job->to, $job->subject, $job->body);
    }
}
```

```php
<?php
declare(strict_types=1);

// === Outbox pattern: garantia transacional ===
final readonly class CreateOrderHandler
{
    public function __construct(
        private PDO $pdo,
        private OrderRepository $orders,
    ) {}

    public function handle(CreateOrderCommand $cmd): OrderId
    {
        $this->pdo->beginTransaction();
        try {
            $order = Order::open($cmd->customerId, $cmd->items);
            $this->orders->save($order);

            // Mesmo transaction: grava evento na outbox
            $stmt = $this->pdo->prepare(
                'INSERT INTO outbox (id, aggregate_id, event_type, payload, created_at)
                 VALUES (:id, :aggId, :type, :payload, NOW())',
            );
            $stmt->execute([
                'id'      => bin2hex(random_bytes(16)),
                'aggId'   => $order->id()->toString(),
                'type'    => 'OrderPlaced',
                'payload' => json_encode(['orderId' => $order->id()->toString(), 'total' => $order->total()]),
            ]);

            $this->pdo->commit();
            return $order->id();
        } catch (\Throwable $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }
}

// Worker separado lê outbox e publica em broker; idempotência via outbox.id
final class OutboxDispatcher
{
    public function tick(): void
    {
        $rows = $this->pdo->query(
            'SELECT id, event_type, payload FROM outbox
             WHERE dispatched_at IS NULL ORDER BY created_at LIMIT 100',
        )->fetchAll(PDO::FETCH_ASSOC);

        foreach ($rows as $row) {
            $this->broker->publish($row['event_type'], $row['payload']);
            $this->pdo->prepare('UPDATE outbox SET dispatched_at = NOW() WHERE id = ?')
                      ->execute([$row['id']]);
        }
    }
}
```

```yaml
# Symfony Messenger — configuração típica
framework:
    messenger:
        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'  # amqp://... ou redis://...
                retry_strategy:
                    max_retries: 5
                    multiplier: 2
                    max_delay: 3600000
                    delay: 1000
                options:
                    queue_name: 'jobs.high'
            failed:
                dsn: 'doctrine://default?queue_name=failed'
        routing:
            App\Message\SendOrderEmail: async
            App\Message\GenerateInvoice: async
        failure_transport: failed
        default_bus: messenger.bus.default
```

```bash
# Symfony Messenger workers
php bin/console messenger:consume async --limit=1000 --time-limit=3600 --memory-limit=128M

# Laravel Queue
php artisan queue:work redis --queue=high,default --tries=5 --max-jobs=1000 --max-time=3600

# Em produção: supervisor/systemd reinicia ao morrer
# Sempre limitar --max-jobs e --time-limit para evitar memory leak
```

## Quando usar / quando não usar

- **Sempre que operação >100ms** que pode ser assíncrona sem prejudicar UX (envio de e-mail, processamento de imagem, geração de PDF, sync com sistema externo).
- **Sempre que há efeito externo flaky** — APIs de terceiros, SMTP, webhooks. Isolar com fila evita falhas em cascata.
- **Eventos de domínio** consumidos por bounded contexts diferentes — não comprometer o write transaction principal.
- **Cargas em batch** — processamento noturno, relatórios — naturalmente assíncrono.
- **Redis Lists/Streams** — começo simples, stack já tem Redis, sem necessidade de routing complexo.
- **RabbitMQ** — routing rico (topic exchanges, header exchanges), prioridade nativa, DLX, ack manual sofisticado, multi-tenancy.
- **Kafka** — event streaming, replay (consumers podem reler do offset zero), pipelines de dados, integração entre sistemas heterogêneos com produtores/consumidores em linguagens diferentes.
- **SQS** — quando você quer zero manutenção e está em AWS.
- **Não use** filas para fluxo crítico síncrono de UX onde resposta imediata é esperada (login, autenticação, validação de formulário). Aí cache > fila.
- **Não use** Kafka como "fila de jobs" simples — overkill, configuração complexa, custo operacional alto.
- **Cuidado** com filas para operações com SLA rígido — se broker fica fora 1h, jobs param 1h.

## Armadilhas comuns

1. **Worker sem reinício automático.** Worker crasha (OOM, exception não tratada, SIGSEGV), ninguém percebe, fila cresce silenciosamente até alertar dias depois. *Por quê*: ausência de supervisor/systemd com `Restart=always`. Sempre rodar workers sob processo de supervisão.

2. **Memory leak em worker long-running.** Worker processa 100k jobs, RSS sobe de 50MB para 4GB, OOM kill. *Por quê*: PHP não libera memória de objetos com referências circulares automaticamente, ORMs mantêm identity maps. Mitigação: `--max-jobs=1000`, `--memory-limit=128M`, `gc_collect_cycles()` periódico, reset de EntityManager.

3. **Esquecer DLQ.** Mensagem "venenosa" (payload inválido, edge case) entra em loop infinito de retry, log enche, worker fica preso em uma mensagem ruim. *Por quê*: confiança excessiva no produtor; sempre limitar tentativas e mover para DLQ.

4. **Consumer não-idempotente em broker at-least-once.** Mesma mensagem reprocessada após network glitch entre worker e broker (ack se perdeu) → e-mail duplicado, cobrança dupla. *Por quê*: assumir exactly-once quando broker garante at-least-once. Sempre projete consumer para tolerar reentrega.

5. **Payload gigante na fila.** Mensagem com 5MB de payload (imagem inteira), Redis vira gargalo, latência geral degrada. *Por quê*: confusão entre "passar dados" e "passar referência". Passe **id** (`{"orderId": 42}`), worker busca dados frescos do banco.

6. **Visibility timeout mal calibrado.** Worker leva 90s para processar job, visibility timeout é 60s — broker reentrega para outro worker, dois processam a mesma mensagem em paralelo. *Por quê*: timeout não bate com pior caso real de processamento. Meça p99 do handler e configure timeout = p99 × 3.

7. **Mistura de prioridades em uma fila só.** Job de e-mail crítico fica atrás de 50k jobs de "recalcular ranking semanal". *Por quê*: ausência de filas separadas por prioridade. Use `jobs.high`, `jobs.default`, `jobs.low` e workers dedicados ou ordem de consumo.

8. **`serialize()` com objetos PSR-3 logger, container, conexão.** Payload contém referência a serviço, falha ou serializa coisa imensa. *Por quê*: descuido — job deve carregar **dados**, não serviços. Use DTOs imutáveis simples como payload.

## Links relacionados

- [[Arquitetura/Domain-Events-CQRS]]
- [[Cache-PSR-6-16]]
- [[FrankenPHP-RoadRunner-Swoole]]
- [[APIs/Versionamento-Idempotencia]]
- [[Fibers-N1]]
- [[Profiling]]

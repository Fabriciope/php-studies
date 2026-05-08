---
tags: [php, performance, filas, async]
fase: 6
status: stub
---

# Filas (Redis / RabbitMQ)

## Conceito

Fila é um buffer entre **produtor** (rota HTTP) e **consumidor** (worker em loop). Produtor enfileira mensagem e responde; worker processa em background. Decora o sistema com **resiliência** (retry, DLQ) e **escala independente** (crescer worker sem crescer web).

Opções comuns:
- **Redis** + lib (Predis, php-redis): simples, rápido, sem features ricas (Lists ou Streams).
- **RabbitMQ**: maduro, roteamento avançado (exchanges), DLX, prioridade, ack manual.
- **Kafka**: log distribuído, ordem por partição, replay. Overkill para "envia e-mail", essencial para eventos.

## Por que importa

HTTP request com 8s para enviar e-mail SMTP é UX terrível e SLA ruim. Empurra para fila → resposta em 50ms; o e-mail vai quando o worker pegar. Também isola falhas: SMTP fora não derruba checkout.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

// produtor — enfileira na transação do agregado (outbox simplificado)
interface JobQueue {
    public function push(string $queue, object $job): void;
}

final readonly class RedisJobQueue implements JobQueue {
    public function __construct(private \Redis $redis) {}

    public function push(string $queue, object $job): void {
        $this->redis->rPush(
            "queue:{$queue}",
            serialize(['class' => $job::class, 'data' => $job, 'attempts' => 0]),
        );
    }
}

// worker com retry exponencial
final class Worker {
    public function __construct(
        private \Redis $redis,
        private LoggerInterface $log,
        private int $maxAttempts = 5,
    ) {}

    public function run(string $queue): never {
        while (true) {
            $raw = $this->redis->blPop(["queue:{$queue}"], 5);
            if (!$raw) continue;
            $msg = unserialize($raw[1]);

            try {
                $this->dispatch($msg['data']);
            } catch (\Throwable $e) {
                $msg['attempts']++;
                if ($msg['attempts'] >= $this->maxAttempts) {
                    $this->redis->rPush("queue:{$queue}:dead", serialize($msg));
                    $this->log->error('job.dead', ['class' => $msg['class'], 'err' => $e->getMessage()]);
                    continue;
                }
                $delay = min(60 * (2 ** $msg['attempts']), 3600); // exponential backoff
                $this->redis->zAdd("queue:{$queue}:delayed", time() + $delay, serialize($msg));
            }
        }
    }
}
```

## Padrões de retry

- **Exponential backoff**: `delay = min(base * 2^attempt, max_cap)`.
- **Jitter**: somar aleatório para evitar thundering herd.
- **DLQ (dead-letter queue)**: depois de N tentativas, mensagem vai para fila morta (manual review).
- **Idempotência do consumidor**: workers podem processar a mesma mensagem 2x (broker garante at-least-once). Job precisa ser idempotente.

## Quando usar / quando não usar

- **Sempre que a operação >100ms** ou tem efeito externo flaky (e-mail, SMS, webhook).
- **Eventos de domínio** consumidos por outro contexto.
- **Não usar** para transação síncrona crítica de fluxo único (login).
- **Redis vs RabbitMQ**: comece com Redis se a stack já tem; suba para RabbitMQ se precisa de prioridade, fanout, DLX nativo.

## Armadilhas comuns

- Worker sem reinício automático (systemd, supervisor) — morre, fila cresce, alerta surdo.
- `serialize`/`unserialize` de objeto com referência circular — payload corrompido.
- Mensagem grande na fila — Redis vira gargalo. Passe **id** e leia detalhes do banco.
- Esquecer DLQ — mensagem ruim entra em loop infinito de retry.

## Links relacionados

- [[Arquitetura/Domain-Events-CQRS]]
- [[Cache-PSR-6-16]]
- [[FrankenPHP-RoadRunner-Swoole]]

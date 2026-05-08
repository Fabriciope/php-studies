---
tags: [roadmap, php, planejamento]
fase: 0
status: stub
---

# 01 · Roadmap — 16 semanas

Carga: 15–20 h/semana. Cada fase fecha com **um entregável obrigatório**. Sem o entregável, não passa.

## Visão geral

| Fase | Sem | Tema                          | Entregável                                 |
| ---- | --- | ----------------------------- | ------------------------------------------ |
| 1    | 1–2 | Fundamentos                   | Mini-lib de validação tipada               |
| 2    | 3–4 | OOP moderno (8.1→8.4)         | Lib de Value Objects                       |
| 3    | 5–6 | SOLID + padrões               | Pipeline de pedidos com 5 padrões          |
| 4    | 7–9 | Arquitetura (DDD/Clean/Hex)   | Módulo de pedidos hexagonal                |
| 5    | 10–11 | APIs maduras + PSRs         | API REST sem framework + OpenAPI           |
| 6    | 12–13 | Performance & runtime       | Relatório FPM × FrankenPHP × RoadRunner    |
| 7    | 14  | Infra & qualidade             | Docker multi-stage + CI verde + PHPStan 8  |
| 8    | 15–16 | Projeto integrador          | Sistema de pedidos completo + ADRs         |

## Detalhe semanal

### Semana 1 — strict + tipos
- Ler [[Fundamentos/Strict-Types]] e [[Fundamentos/Sistema-de-Tipos]]
- Refatorar 5 funções legadas para tipagem total
- Comparar saída strict vs coercive em 10 casos

### Semana 2 — exceções + ini
- [[Fundamentos/Hierarquia-Throwable]] e [[Fundamentos/Configuracao-PHP-INI]]
- Hierarquia própria de exceções de domínio
- Auditoria de um `php.ini` real
- ✅ Fechar **Entregável 1**

### Semana 3 — enums + readonly
- [[OOP-Moderno/Enums]], [[OOP-Moderno/Readonly-Classes]]
- Substituir 3 conjuntos de constantes por Enums
- Migrar DTO mutável → readonly class

### Semana 4 — hooks + atributos
- [[OOP-Moderno/Property-Hooks]], [[OOP-Moderno/Asymmetric-Visibility]], [[OOP-Moderno/Constructor-Promotion]], [[OOP-Moderno/Atributos]]
- Property hook com validação lazy
- Atributo customizado lido via Reflection
- ✅ Fechar **Entregável 2**

### Semana 5 — SOLID + padrões criação/comportamento
- [[Padroes/SOLID]], [[Padroes/Strategy]], [[Padroes/Factory]]
- Strategy de cálculo de frete (3 transportadoras)

### Semana 6 — padrões estruturais + DI
- [[Padroes/Repository]], [[Padroes/Decorator]], [[Padroes/Observer]], [[Padroes/Specification]]
- Repository com 2 backends, Decorator de cache
- Container PSR-11 minúsculo
- ✅ Fechar **Entregável 3**

### Semana 7 — camadas + hexagonal
- [[Arquitetura/Camadas-DDD]], [[Arquitetura/Hexagonal]]
- Modelar domínio do pedido sem framework

### Semana 8 — VO + aggregates + use cases
- [[Arquitetura/Value-Objects]], [[Arquitetura/Aggregates]], [[Arquitetura/Use-Cases]]
- Use Case "Pagar pedido" com porta de pagamento
- Adapter Stripe + adapter fake

### Semana 9 — domain events + CQRS
- [[Arquitetura/Domain-Events-CQRS]]
- Publicar e consumir Domain Event
- ✅ Fechar **Entregável 4**

### Semana 10 — PSRs + REST maduro
- [[APIs/PSRs]], [[APIs/REST-Maduro]]
- Middleware PSR-15 de logging estruturado

### Semana 11 — paginação + idempotência + auth + OpenAPI
- [[APIs/Paginacao-Cursor]], [[APIs/Versionamento-Idempotencia]], [[APIs/JWT-vs-Sessoes]], [[APIs/OpenAPI]]
- Endpoint POST idempotente, paginação cursor sobre 1M linhas
- ✅ Fechar **Entregável 5**

### Semana 12 — OPcache, JIT, profiling
- [[Performance/OPcache-JIT-Preloading]], [[Performance/Profiling]]
- Profile com SPX e otimizar hotpath

### Semana 13 — cache, filas, runtimes, fibers
- [[Performance/Cache-PSR-6-16]], [[Performance/Filas]], [[Performance/FrankenPHP-RoadRunner-Swoole]], [[Performance/Fibers-N1]]
- Benchmark FPM × FrankenPHP, worker de filas com retry
- ✅ Fechar **Entregável 6**

### Semana 14 — infra
- [[Infra/Docker-FPM]], [[Infra/CI-CD]], [[Infra/PHPStan-Rector]], [[Infra/Observabilidade-OWASP]]
- ✅ Fechar **Entregável 7**

### Semanas 15–16 — projeto integrador
- Catálogo + carrinho + checkout
- Strategy de descontos, máquina de estados, Domain Events, fila, JWT, cache, OpenAPI
- README com ADRs, vídeo de defesa de 10 min
- ✅ Fechar **Entregável 8 — portfólio**

## Regras

- **Não pular fase.** Mesmo que o tema pareça conhecido — o entregável força a executar.
- **Revisão na sexta de cada semana** (loop 2 do método).
- **Defesa em voz alta** ao fechar cada fase (loop 3).

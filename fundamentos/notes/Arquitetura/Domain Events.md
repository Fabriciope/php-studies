---
tags: [php, arquitetura, fase_5]
fase: 5
status: stub
---

# Domain Events

## Conceito

Domain Events é um ponto de estudo de Arquitetura que deve ser entendido pelo comportamento observado no runtime, pelo impacto no design e pela forma idiomática de uso em PHP 8.3/8.4.

## Por que importa

Importa porque reduz ambiguidade, melhora manutenção e cria uma base segura para evoluir de scripts simples para aplicações com Composer, testes, análise estática e arquitetura.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

final readonly class OrderPaid { public function __construct(public string $orderId) {} }
```

## Quando usar / quando não usar

Use quando o conceito tornar o código mais explícito, testável ou previsível. Não use como cerimônia: se ele esconder regra simples, acoplar detalhes ou dificultar leitura, prefira a solução direta.

## Armadilhas comuns

- Decorar a sintaxe sem entender coerção, erros e limites do runtime.
- Misturar responsabilidade de domínio, infraestrutura e apresentação.
- Ignorar `strict_types=1`, autoload PSR-4 ou validação com ferramentas como PHPStan.

## Links relacionados

[[Aggregates]] [[CQRS]] [[Use Cases]] [[Value Objects]] [[Repository]] [[01-Roadmap]]

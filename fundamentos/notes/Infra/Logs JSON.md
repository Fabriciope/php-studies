---
tags: [php, infra, fase_7]
fase: 7
status: stub
---

# Logs JSON

## Conceito

Logs JSON é um ponto de estudo de Infra que deve ser entendido pelo comportamento observado no runtime, pelo impacto no design e pela forma idiomática de uso em PHP 8.3/8.4.

## Por que importa

Importa porque reduz ambiguidade, melhora manutenção e cria uma base segura para evoluir de scripts simples para aplicações com Composer, testes, análise estática e arquitetura.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

echo json_encode(['level' => 'info', 'event' => 'order.created']);
```

## Quando usar / quando não usar

Use quando o conceito tornar o código mais explícito, testável ou previsível. Não use como cerimônia: se ele esconder regra simples, acoplar detalhes ou dificultar leitura, prefira a solução direta.

## Armadilhas comuns

- Decorar a sintaxe sem entender coerção, erros e limites do runtime.
- Misturar responsabilidade de domínio, infraestrutura e apresentação.
- Ignorar `strict_types=1`, autoload PSR-4 ou validação com ferramentas como PHPStan.

## Links relacionados

[[Rector]] [[OpenTelemetry]] [[Docker multi-stage]] [[PHPStan nível 8]] [[OpenTelemetry]] [[01-Roadmap]]

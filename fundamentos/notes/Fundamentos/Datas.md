---
tags: [php, fundamentos, fase_5]
fase: 5
status: stub
---

# Datas

## Conceito

Datas devem ser tratadas com objetos como `DateTimeImmutable`, timezone explícito e formatação controlada.

## Por que importa

Data e hora parecem simples, mas envolvem fuso, formato de entrada, comparação e imutabilidade.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

$timezone = new DateTimeZone('America/Sao_Paulo');
$createdAt = new DateTimeImmutable('now', $timezone);
$expiresAt = $createdAt->modify('+7 days');

echo $expiresAt->format(DateTimeInterface::ATOM);
```

## Quando usar / quando não usar

Use `DateTimeImmutable` para evitar alteração acidental. Não compare datas como strings sem formato ordenável e timezone conhecido.

## Armadilhas comuns

- Usar timezone padrão sem saber qual é.
- Mutar `DateTime` compartilhado.
- Aceitar formato de data de usuário sem parse e validação.

## Links relacionados

[[php.ini]] [[Erros e exceções]] [[Sistema de tipos]]

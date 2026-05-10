---
tags: [php, fundamentos, fase_4]
fase: 4
status: stub
---

# Strings

## Conceito

Strings representam texto e bytes. Em aplicações reais, elas carregam nomes, e-mails, JSON, caminhos, mensagens e conteúdo de arquivos.

## Por que importa

Manipular strings sem atenção a encoding, espaços e validação causa bugs em entrada de usuário, slugs, buscas e integrações.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

$title = '  PHP Moderno Básico  ';
$normalized = trim($title);
$slug = strtolower(str_replace(' ', '-', $normalized));

echo $slug . PHP_EOL;
```

## Quando usar / quando não usar

Use funções de string para normalização simples e extensões multibyte quando houver acentos ou Unicode. Não trate toda string como dado já validado.

## Armadilhas comuns

- Ignorar espaços no início/fim.
- Usar `strlen()` para texto multibyte quando o tamanho visual importa.
- Concatenar dados externos em mensagens sem validação ou escape no contexto correto.

## Links relacionados

[[Arrays]] [[Variáveis e tipos]] [[Funções]]

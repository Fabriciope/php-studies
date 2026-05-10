---
tags: [php, indice, fundamentos]
fase: 0
status: stub
---

# 00-Index

## Conceito

Índice do vault, agora organizado para priorizar fundamentos profundos de PHP antes de padrões, arquitetura e infraestrutura.

## Por que importa

Ajuda a navegar por uma trilha que começa no runtime da linguagem: tipos, funções, arrays, erros, exceções, arquivos e Composer básico.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

$inicio = ['Sintaxe básica', 'Variáveis e tipos', 'Funções'];
foreach ($inicio as $nota) {
    echo "Estude: {$nota}" . PHP_EOL;
}
```

## Quando usar / quando não usar

Use como ponto de partida e checklist de revisão. Não use para pular direto para notas avançadas sem consolidar as notas de Fundamentos.

## Armadilhas comuns

- Confundir quantidade de notas com domínio real.
- Ler sobre arquitetura antes de prever tipos, retornos e erros de funções simples.
- Não registrar exemplos próprios após executar código no CLI.

## Links relacionados

[[01-Roadmap]] [[Sintaxe básica]] [[Variáveis e tipos]] [[Funções]] [[Arrays]] [[Erros e exceções]] [[O que é Composer]]

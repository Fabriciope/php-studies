---
tags: [php, fundamentos, fase_1]
fase: 1
status: stub
---

# Sintaxe básica

## Conceito

Sintaxe básica é o conjunto de regras para escrever arquivos PHP: abertura `<?php`, instruções terminadas por ponto e vírgula, blocos com chaves, comentários e expressões.

## Por que importa

Sem sintaxe confortável, o estudo fica preso a erros mecânicos. A meta é reconhecer rapidamente se uma falha veio da gramática do PHP ou da lógica do programa.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

// Comentário de linha: útil para intenção, não para repetir o óbvio.
$name = 'Ada';
$age = 37;

echo "{$name} tem {$age} anos." . PHP_EOL;
```

## Quando usar / quando não usar

Use scripts pequenos para treinar leitura e escrita. Não misture HTML, banco e framework enquanto ainda houver dúvida sobre variáveis, blocos e saída.

## Armadilhas comuns

- Esquecer `;` no fim de instruções.
- Abrir `<?php` várias vezes sem necessidade em arquivo PHP puro.
- Confundir erro de sintaxe com exceção tratável.

## Links relacionados

[[Variáveis e tipos]] [[Controle de fluxo]] [[php.ini]] [[01-Roadmap]]

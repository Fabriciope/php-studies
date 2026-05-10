---
tags: [php, fundamentos, fase_5]
fase: 5
status: stub
---

# Erros e exceções

## Conceito

Erros são falhas reportadas pelo runtime; exceções são objetos lançados para interromper o fluxo normal e exigir tratamento explícito.

## Por que importa

Saber quando falhar, quando capturar e que mensagem expor separa código previsível de scripts silenciosamente quebrados.

## Exemplo de código PHP

```php
<?php declare(strict_types=1);

function readRequiredFile(string $path): string
{
    if (! is_file($path)) {
        throw new RuntimeException("Arquivo não encontrado: {$path}");
    }

    return file_get_contents($path) ?: '';
}
```

## Quando usar / quando não usar

Use exceções para estados inválidos que impedem a operação. Não capture `Throwable` apenas para esconder erro e continuar com dado inconsistente.

## Armadilhas comuns

- Capturar exceção e não registrar nada.
- Lançar `Exception` genérica para todos os casos.
- Confundir warning recuperável com resultado válido.

## Links relacionados

[[Throwable]] [[Funções]] [[php.ini]]

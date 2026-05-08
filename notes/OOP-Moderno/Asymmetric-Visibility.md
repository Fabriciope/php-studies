---
tags: [php, oop, php84]
fase: 2
status: stub
---

# Asymmetric Visibility (PHP 8.4)

## Conceito

Permite definir visibilidades **diferentes** para leitura e escrita de uma propriedade. A leitura segue o modificador padrão (`public`); a escrita usa um segundo modificador entre parênteses: `public private(set)`, `public protected(set)`.

## Por que importa

Resolve o caso clássico "quero expor o valor mas só a própria classe pode mudar". Antes você caía em `public function getX()` + `private $x` — boilerplate. Agora a propriedade é honesta: lê pública, escreve privada.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

final class Counter {
    public private(set) int $value = 0;

    public function increment(): void {
        $this->value++;        // ok, dentro da classe
    }
}

$c = new Counter();
echo $c->value;       // 0   — leitura pública
$c->increment();
echo $c->value;       // 1
$c->value = 99;       // Error: Cannot modify private(set) property from global scope

// herança: protected(set) deixa subclasse escrever
abstract class Entity {
    public protected(set) string $id;
}
```

## Quando usar / quando não usar

- **Entidades de domínio** com estado controlado: `id`, `createdAt`, contadores.
- **DTOs com inicialização parcial** que depois ficam imutáveis a partir de fora.
- **Não substitui readonly** quando a propriedade nunca muda após construção — readonly é mais forte.
- **Não usar como atalho para getter** se a propriedade de fato precisa ser computada — use property hook.

## Armadilhas comuns

- A regra de visibilidade de `set` precisa ser **mais restritiva ou igual** à de leitura. `private public(set)` não existe.
- `readonly` é estritamente mais forte que `public private(set)` — escolha por intenção: readonly = "nunca muda", asymmetric = "só eu mudo".
- `clone` respeita visibilidade do `set` da mesma forma.

## Links relacionados

- [[Readonly-Classes]]
- [[Property-Hooks]]
- [[Constructor-Promotion]]

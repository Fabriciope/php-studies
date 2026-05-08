---
tags: [php, oop, php84]
fase: 2
status: stub
---

# Property Hooks (PHP 8.4)

## Conceito

PHP 8.4 traz `get`/`set` hooks em propriedades — equivalentes a propriedades computadas em C# ou Kotlin. Permite encapsular leitura e escrita sem virar getter/setter explícito, e mantém a sintaxe de propriedade (`$obj->name`) no caller.

## Por que importa

Antes, validar uma escrita exigia `setName()`/`getName()` — verboso e fácil de "burlar" expondo a propriedade. Com hooks, o invariante mora **na propriedade**, não em método paralelo. É a forma idiomática de encapsular sem boilerplate.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

final class User {
    public string $email {
        set(string $value) {
            $value = strtolower(trim($value));
            if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
                throw new InvalidArgumentException('invalid email');
            }
            $this->email = $value;
        }
    }

    // hook só de leitura — propriedade derivada
    public string $domain {
        get => substr($this->email, (int) strpos($this->email, '@') + 1);
    }

    public function __construct(string $email) {
        $this->email = $email;   // dispara o set hook
    }
}

$u = new User('  Foo@Bar.COM ');
echo $u->email;   // foo@bar.com
echo $u->domain;  // bar.com
```

## Quando usar / quando não usar

- **Validação de invariante** que mora numa única propriedade.
- **Propriedades derivadas** (cálculo barato) — vira `get` puro.
- **Não usar** para lógica pesada — `get` é chamado a cada leitura. Cache em outra propriedade se necessário.
- **Não usar** quando a classe deveria ser readonly — hook de set não combina com imutabilidade; prefira validação no construtor.

## Armadilhas comuns

- Atribuir `$this->email = $value` **dentro** do `set` hook **não** dispara recursão — é o storage real.
- Sem `set`, a propriedade é só leitura externa **mas** ainda escrevível em `__construct`. Para realmente imutável, combine com `readonly`.
- Property hooks em propriedades tipadas seguem strict_types: `set(string $v)` rejeita `int` em strict.

## Links relacionados

- [[Readonly-Classes]]
- [[Asymmetric-Visibility]]
- [[Arquitetura/Value-Objects]]

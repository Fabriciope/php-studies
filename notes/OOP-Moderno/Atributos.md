---
tags: [php, oop, atributos, reflection]
fase: 2
status: stub
---

# Atributos (Attributes)

## Conceito

Atributos (PHP 8.0+) são metadados em sintaxe nativa, escritos com `#[Foo(...)]`. São lidos via Reflection. Substituem docblocks `@Annotation` que dependiam de parser externo (Doctrine Annotations).

## Por que importa

Permitem declarar comportamento **declarativamente** — validação, rotas, mapeamento ORM, serialização — mantendo a classe POPO. Como são objetos PHP normais, têm tipo, autocomplete, refactor seguro. Adeus comentários mágicos.

## Exemplo de código PHP

```php
<?php
declare(strict_types=1);

#[Attribute(Attribute::TARGET_PROPERTY | Attribute::IS_REPEATABLE)]
final readonly class Validate {
    public function __construct(public string $rule, public mixed $arg = null) {}
}

final class CreateUser {
    #[Validate('required')]
    #[Validate('max', 120)]
    public string $name;

    #[Validate('email')]
    public string $email;
}

// reader genérico
function rulesOf(string $class): array {
    $rules = [];
    foreach ((new ReflectionClass($class))->getProperties() as $prop) {
        foreach ($prop->getAttributes(Validate::class) as $attr) {
            $rules[$prop->getName()][] = $attr->newInstance();
        }
    }
    return $rules;
}

$rules = rulesOf(CreateUser::class);
// ['name' => [Validate('required'), Validate('max', 120)], 'email' => [...]]
```

## Quando usar / quando não usar

- **Validação declarativa, rotas, DI hints, mapping** — atributo é a ferramenta certa.
- **Mensagens cross-cutting** que pertencem ao tipo, não ao caller.
- **Não usar** quando a configuração precisa mudar em runtime (atributos são estáticos).
- **Não abusar** — se você precisa de 6 atributos para uma classe simples, talvez o problema seja a classe.

## Armadilhas comuns

- Atributo sem `#[Attribute]` na classe alvo é apenas metadado solto — Reflection retorna, mas `newInstance()` falha.
- `Attribute::IS_REPEATABLE` é necessário se o mesmo atributo aparece duas vezes no mesmo alvo.
- Reflection custa: cachear o resultado de `rulesOf()` em produção (preload + container compilado).

## Links relacionados

- [[Readonly-Classes]]
- [[Sistema-de-Tipos]]
- [[Padroes/SOLID]]

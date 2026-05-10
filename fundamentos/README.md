# Fundamentos de PHP Moderno

Este guia foi ajustado para focar menos em tópicos avançados cedo demais e mais em uma base realmente sólida de PHP 8.3/8.4: tipos, coerção, funções, arrays, strings, datas, erros, exceções, arquivos, JSON, namespaces e Composer básico.

A ideia é formar domínio de linguagem antes de arquitetura, frameworks, filas, containers ou infraestrutura.

## Como abrir a página

Abra `fundamentos/index.html` diretamente no navegador ou rode um servidor local a partir da raiz do repositório:

```bash
php -S localhost:8000
```

Depois acesse:

```text
http://localhost:8000/fundamentos/index.html
```

## Como importar o vault no Obsidian

1. Abra o Obsidian.
2. Escolha **Open folder as vault**.
3. Selecione a pasta `fundamentos/notes/`.
4. Comece por `00-Index.md`, `01-Roadmap.md` e pelas notas da pasta `Fundamentos/`.

## Como estudar o roadmap

O roadmap tem 16 semanas, estimando 15-20h por semana. A prioridade é consolidar comportamento do PHP em scripts pequenos antes de avançar para organização com Composer e objetos.

A ordem recomendada é:

1. Ambiente, CLI e sintaxe.
2. Tipos, coerção e `strict_types`.
3. Controle de fluxo e funções.
4. Arrays, strings e dados estruturados.
5. Datas, arquivos, JSON, erros e exceções.
6. Composer, namespaces e autoload PSR-4.
7. PHP moderno aplicado com classes simples, readonly e enums.
8. Projeto integrador de fundamentos.

## Como usar Composer durante os estudos

Composer deve ser usado como ferramenta de organização, não como atalho para evitar entender PHP.

Comandos úteis:

```bash
php -v
composer -V
composer init
composer dump-autoload
php -S localhost:8000
```

Comece com scripts isolados. Depois transforme exercícios em um projeto com:

```text
src/
public/
composer.json
```

Use autoload PSR-4 quando já houver classes ou funções organizadas em arquivos separados.

## Como evoluir os stubs das notas

Cada nota começa com `status: stub`. Ao estudar, atualize principalmente as notas de `Fundamentos/` com:

- uma definição curta;
- um exemplo PHP executado localmente;
- uma armadilha real que você encontrou;
- o tipo de entrada e saída das funções;
- links para notas relacionadas usando wiki-links, como `[[Sistema de tipos]]`, `[[Funções]]` e `[[strict_types]]`.

Quando uma nota estiver madura, altere para `status: draft` ou `status: evergreen`.

## Como executar exemplos PHP localmente

Crie um arquivo temporário e declare tipos estritos:

```php
<?php declare(strict_types=1);

function double(int $value): int
{
    return $value * 2;
}

var_dump(double(21));
```

Execute com:

```bash
php exemplo.php
```

Para exemplos com autoload:

1. configure `autoload.psr-4` no `composer.json`;
2. rode `composer dump-autoload`;
3. inclua `require __DIR__ . '/vendor/autoload.php';` no ponto de entrada.

## O que deixar para depois

Arquitetura, PSRs avançadas, performance, filas, Docker e observabilidade continuam no vault como apêndices. Eles são importantes, mas só devem ganhar prioridade depois que tipos, funções, arrays, erros, Composer e classes simples estiverem naturais.

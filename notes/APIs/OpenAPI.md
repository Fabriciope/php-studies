---
tags: [php, api, openapi, contrato]
fase: 5
status: stub
---

# OpenAPI 3.1

## Conceito

OpenAPI é uma especificação (YAML/JSON) que descreve API HTTP de forma machine-readable. Define paths, schemas, autenticação, exemplos, erros. Da spec geram-se: docs (Swagger UI/Redoc), clientes em N linguagens, mocks, contract tests.

## Por que importa

Sem contrato escrito, "como funciona o endpoint X" vira tribal knowledge. OpenAPI vira a **fonte da verdade** entre back, front e parceiros. CI valida que código bate com spec — quebra a "doc desatualizada".

Em equipes maduras: **spec-first** (a spec nasce primeiro, código se adapta) ou **code-first** (atributos PHP geram a spec). Escolher depende do contexto.

## Exemplo de código PHP

```yaml
# openapi.yaml — spec mínima
openapi: 3.1.0
info:
  title: Orders API
  version: 1.0.0
servers:
  - url: https://api.example.com/v1
paths:
  /orders:
    post:
      summary: Cria pedido
      parameters:
        - in: header
          name: Idempotency-Key
          required: true
          schema: { type: string, format: uuid }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/CreateOrder' }
      responses:
        '201':
          description: Criado
          headers:
            Location: { schema: { type: string } }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Order' }
        '422': { $ref: '#/components/responses/Problem' }
components:
  schemas:
    Order:
      type: object
      required: [id, status, total]
      properties:
        id: { type: string, format: uuid }
        status: { type: string, enum: [pending, paid, shipped] }
        total: { type: integer, description: cents }
```

```php
<?php
// code-first: gerar spec a partir de atributos
#[Route('/orders', methods: ['POST'])]
#[RequestBody(CreateOrderRequest::class)]
#[Response(201, OrderResource::class)]
#[ProblemResponse(422)]
final readonly class CreateOrderController { /* ... */ }
```

## Quando usar / quando não usar

- **Sempre** em API com >1 consumidor ou parceiro externo.
- **Spec-first** em times grandes ou contratos com integração externa estável.
- **Code-first** quando o time é pequeno e o contrato evolui rápido junto do código.
- **Não usar** em script CLI ou worker — não tem superfície HTTP.

## Armadilhas comuns

- Spec gerada e nunca validada contra produção — ela mente. Use ferramenta de **contract testing** (Schemathesis, Dredd) no CI.
- `additionalProperties: true` por default — aceita qualquer campo. Force `false` em modelos críticos.
- Esquemas duplicados em vez de `$ref` — quando muda um, esquece o outro.
- Documentar erro genérico "500 — internal error" sem usar `application/problem+json`. Padronize erros via [[REST-Maduro]].

## Links relacionados

- [[REST-Maduro]]
- [[PSRs]]
- [[Infra/CI-CD]]

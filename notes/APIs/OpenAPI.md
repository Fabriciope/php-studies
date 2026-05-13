---
tags: [php, api, openapi, contrato]
fase: 5
status: draft
---

# OpenAPI 3.1

## Conceito

OpenAPI é uma especificação aberta (YAML ou JSON) que descreve uma API HTTP de forma machine-readable. Nasceu como **Swagger** em 2010 (Wordnik), foi doada à Linux Foundation em 2015 e renomeada OpenAPI Specification (OAS). Hoje é mantida pela **OpenAPI Initiative**, com membros incluindo Google, Microsoft, IBM, Stripe, Postman.

A spec descreve, em um único arquivo (ou conjunto modular):

- **Metadados** (`info`): título, versão, descrição, contato, licença.
- **Servers**: URLs base (produção, staging, sandbox) com possíveis variáveis.
- **Paths**: operações HTTP (GET, POST, PUT, etc.) por endpoint, com parâmetros, request body, responses e exemplos.
- **Components**: schemas (JSON Schema), parâmetros, responses, examples, request bodies, headers, security schemes — todos reutilizáveis via `$ref`.
- **Security**: schemes (Bearer, OAuth2, API Key, OpenID Connect) e quais operações exigem cada um.
- **Tags**: agrupamento lógico para a renderização da documentação.
- **Webhooks** (OAS 3.1): eventos emitidos pelo servidor.

### OpenAPI 3.0 vs 3.1

Versão 3.0.0 saiu em 2017; 3.1.0 em 2021. A diferença mais consequente: **3.1 é 100% compatível com JSON Schema draft 2020-12**. Antes, OpenAPI 3.0 usava um *dialeto* de JSON Schema com diferenças sutis (`nullable: true` em vez de `type: ["string", "null"]`, sem suporte a `prefixItems`, sem `unevaluatedProperties`). Resultado: ferramentas de validação JSON Schema "puras" não conseguiam validar schemas OpenAPI 3.0 sem adapter.

Outras mudanças 3.1:
- Suporte oficial a **webhooks** (servidor descreve callbacks que envia).
- Tipos `null` via `type: ["string", "null"]` em vez de `nullable: true`.
- `examples` (plural) preferido sobre `example` (singular).
- `info.summary` adicional ao `info.title`.

A maioria das libs ainda suporta os dois; novos projetos devem ir direto para 3.1.

### Composição de schemas: oneOf, allOf, anyOf

JSON Schema oferece três combinadores:

- **allOf** — schema deve satisfazer TODOS os sub-schemas. Útil para herança/composição (`Pet` allOf `BaseEntity` + extras).
- **oneOf** — exatamente UM sub-schema deve casar. Use para uniões discriminadas (`Payment` é `CreditCard` OR `Pix` OR `Boleto`).
- **anyOf** — pelo menos UM deve casar. Mais raro; útil para validação "qualquer destas formas é aceitável".

Para `oneOf` com discriminador, OpenAPI tem campo `discriminator` que aponta para propriedade (`type`) e mapeia valores para schemas. Gera código de cliente mais limpo.

### Spec-first vs Code-first

- **Spec-first**: a spec OpenAPI é editada manualmente (ou via ferramenta como Stoplight, Apicurio) e versionada antes do código. Backend implementa contra ela; clientes geram-se a partir dela. Vantagem: design intencional, revisão de contrato em PR, parceiros podem ver/sugerir antes de existir código. Padrão em times grandes e APIs públicas.
- **Code-first**: a spec é **gerada** a partir do código (atributos PHP 8, docblocks, reflection). Ferramentas: `zircote/swagger-php`, `nelmio/api-doc-bundle` (Symfony), `darkaonline/l5-swagger` (Laravel), API Platform. Vantagem: zero divergência entre spec e código. Desvantagem: design tende a refletir o modelo, não a necessidade do consumidor.

Híbrido moderno: spec-first para o **contrato externo**, code-first para detalhes internos.

### Ferramentas do ecossistema

- **Documentação**: Swagger UI (clássico, interativo), Redoc (mais bonito para docs estáticas), Scalar (moderno, dark mode bom), Rapidoc.
- **Mock servers**: Prism, Mockoon — sobem servidor HTTP que responde conforme a spec.
- **Geração de clientes**: `openapi-generator` (Java, suporta 50+ linguagens), `openapi-typescript-codegen`, `kiota` (Microsoft).
- **Geração de servidor**: stubs Slim/Symfony a partir da spec (menos comum em PHP que em Java/Go).
- **Validação runtime**: `league/openapi-psr7-validator` valida request/response em PHP contra a spec — força no CI.
- **Contract testing**: Schemathesis (gera fuzz tests da spec), Dredd (valida cada exemplo da spec), Pact (consumer-driven).
- **Linting**: Spectral (Stoplight) — regras como "todo endpoint deve ter `operationId`", "schemas em PascalCase".

## Por que importa

Sem contrato escrito, "como funciona o endpoint X" vira tribal knowledge ou documentação Markdown desatualizada. OpenAPI vira a **fonte da verdade** entre back-end, front-end, mobile e parceiros. CI valida que o código bate com a spec; quem trabalha em qualquer lado tem documentação confiável e gerada.

Cenário concreto 1 — **front-end e back-end paralelos**: time mobile precisa começar antes do backend terminar. Com spec-first, front-end gera client e usa mock server (Prism) enquanto backend implementa contra a mesma spec. No dia da integração, "apenas" trocam a URL.

Cenário concreto 2 — **parceiro B2B integrando**: parceiro recebe URL da spec, importa no Postman/Insomnia, gera client em Java/Python/Node em 5 minutos. Sem OpenAPI, perde 2 semanas lendo docs em Confluence e fazendo perguntas no Slack.

Cenário concreto 3 — **regressão silenciosa**: dev mudou campo de `total` para `amount` em uma resposta. Sem contract testing, app mobile crasha em produção. Com Schemathesis no CI, build quebra na PR.

Cenário concreto 4 — **auditoria de segurança**: auditor pede inventário de endpoints, parâmetros sensíveis, esquemas de auth. Sem OpenAPI, alguém varre código por horas. Com OpenAPI, exporta CSV em minutos.

## Exemplo de código PHP

Exemplo 1 — spec OpenAPI 3.1 com Problem Details e schema reuse:

```yaml
openapi: 3.1.0
info:
  title: Orders API
  version: 1.0.0
  summary: Gestão de pedidos
  contact:
    email: api@example.com
servers:
  - url: https://api.example.com/v1
paths:
  /orders:
    post:
      operationId: createOrder
      summary: Cria pedido
      tags: [orders]
      parameters:
        - in: header
          name: Idempotency-Key
          required: true
          schema: { type: string, format: uuid }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrder'
      responses:
        '201':
          description: Criado
          headers:
            Location:
              schema: { type: string, format: uri }
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '409': { $ref: '#/components/responses/Conflict' }
        '422': { $ref: '#/components/responses/Validation' }
      security:
        - bearerAuth: []

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
  schemas:
    CreateOrder:
      type: object
      required: [items]
      additionalProperties: false
      properties:
        items:
          type: array
          minItems: 1
          items:
            $ref: '#/components/schemas/OrderItem'
        notes: { type: [string, "null"], maxLength: 500 }
    OrderItem:
      type: object
      required: [sku, qty]
      additionalProperties: false
      properties:
        sku: { type: string, pattern: "^[A-Z0-9-]+$" }
        qty: { type: integer, minimum: 1, maximum: 999 }
    Order:
      type: object
      required: [id, status, total, items]
      properties:
        id: { type: string, format: uuid }
        status:
          type: string
          enum: [pending, paid, shipped, cancelled]
        total: { type: integer, description: "em centavos" }
        items:
          type: array
          items: { $ref: '#/components/schemas/OrderItem' }
    Problem:
      type: object
      required: [type, title, status]
      properties:
        type: { type: string, format: uri }
        title: { type: string }
        status: { type: integer }
        detail: { type: [string, "null"] }
        instance: { type: [string, "null"] }
  responses:
    Conflict:
      description: Conflito de estado
      content:
        application/problem+json:
          schema: { $ref: '#/components/schemas/Problem' }
    Validation:
      description: Erro de validação
      content:
        application/problem+json:
          schema: { $ref: '#/components/schemas/Problem' }
```

Exemplo 2 — oneOf com discriminador (união de pagamentos):

```yaml
components:
  schemas:
    Payment:
      oneOf:
        - $ref: '#/components/schemas/CreditCardPayment'
        - $ref: '#/components/schemas/PixPayment'
        - $ref: '#/components/schemas/BoletoPayment'
      discriminator:
        propertyName: type
        mapping:
          credit_card: '#/components/schemas/CreditCardPayment'
          pix: '#/components/schemas/PixPayment'
          boleto: '#/components/schemas/BoletoPayment'
    CreditCardPayment:
      type: object
      required: [type, card_token]
      properties:
        type: { type: string, const: credit_card }
        card_token: { type: string }
    PixPayment:
      type: object
      required: [type, pix_key]
      properties:
        type: { type: string, const: pix }
        pix_key: { type: string }
```

Exemplo 3 — code-first PHP com atributos (zircote/swagger-php):

```php
<?php
declare(strict_types=1);

use OpenApi\Attributes as OA;

#[OA\Info(title: 'Orders API', version: '1.0.0')]
#[OA\Server(url: 'https://api.example.com/v1')]
final class OpenApiInfo {}

#[OA\Schema(
    schema: 'Order',
    required: ['id', 'status', 'total'],
    properties: [
        new OA\Property(property: 'id', type: 'string', format: 'uuid'),
        new OA\Property(property: 'status', type: 'string', enum: ['pending', 'paid', 'shipped']),
        new OA\Property(property: 'total', type: 'integer', description: 'em centavos'),
    ],
)]
final readonly class OrderResource
{
    public function __construct(
        public string $id,
        public string $status,
        public int $total,
    ) {}
}

final readonly class OrderController
{
    #[OA\Post(
        path: '/orders',
        operationId: 'createOrder',
        tags: ['orders'],
        security: [['bearerAuth' => []]],
        requestBody: new OA\RequestBody(
            required: true,
            content: new OA\JsonContent(ref: '#/components/schemas/CreateOrder'),
        ),
        responses: [
            new OA\Response(response: 201, description: 'Criado',
                content: new OA\JsonContent(ref: '#/components/schemas/Order')),
            new OA\Response(response: 422, description: 'Validação'),
        ],
    )]
    public function create(/* ... */): ResponseInterface { /* ... */ }
}
```

Exemplo 4 — validação runtime contra a spec (league/openapi-psr7-validator):

```php
<?php
declare(strict_types=1);

use League\OpenAPIValidation\PSR15\ValidationMiddlewareBuilder;
use Psr\Http\Message\{ResponseInterface, ServerRequestInterface};
use Psr\Http\Server\{MiddlewareInterface, RequestHandlerInterface};

final readonly class OpenApiValidationMiddleware implements MiddlewareInterface
{
    public function __construct(private string $specPath) {}

    public function process(
        ServerRequestInterface $req,
        RequestHandlerInterface $next,
    ): ResponseInterface {
        $builder = (new ValidationMiddlewareBuilder())
            ->fromYamlFile($this->specPath);

        // valida request — lanca excecao se nao bate com a spec
        $reqValidator = $builder->getServerRequestValidator();
        $reqValidator->validate($req);

        $res = $next->handle($req);

        // valida response — pega divergencias em desenvolvimento
        if (getenv('APP_ENV') !== 'production') {
            $resValidator = $builder->getResponseValidator();
            $resValidator->validate($req, $res);
        }
        return $res;
    }
}
```

## Quando usar / quando não usar

- **Sempre** em API com mais de um consumidor ou parceiro externo.
- **Sempre** em greenfield de plataforma SaaS — spec será valor agregado para clientes.
- **Spec-first** em times grandes, contratos com integração externa estável, ou compliance que exige aprovação de contrato.
- **Code-first** quando o time é pequeno, a API evolui rápido junto do código, e mudança de contrato é coordenada com cliente.
- **Híbrido**: spec-first no contrato exposto + code-first em endpoints internos.
- **Não use** em script CLI, worker assíncrono, daemon — não há superfície HTTP.
- **Não use** OpenAPI para descrever GraphQL (use SDL do GraphQL) ou gRPC (use Protobuf).
- **Considere alternativas** quando a API é fundamentalmente RPC com tipos ricos: AsyncAPI (eventos), Smithy (AWS), TypeSpec (Microsoft).

## Armadilhas comuns

- **Spec gerada e nunca validada contra runtime**: ela mente em silêncio. Use `league/openapi-psr7-validator` em CI ou em ambiente de stage. Idealmente em produção como middleware no modo "log apenas" para detectar divergências.
- **`additionalProperties: true` por default** (sem declarar): aceita qualquer campo extra, encorajando campos não documentados que viram dependência implícita. Force `additionalProperties: false` em request bodies críticos.
- **Schemas duplicados em vez de `$ref`**: quando muda um, esqueceram do outro. Spec entra em estado inconsistente. Use sempre `$ref` e revise no PR.
- **Documentar `500 — internal error` genérico**: erros internos não devem ser parte do contrato no detalhe, mas o **shape** (Problem Details) sim. Padronize via component reutilizável.
- **Esquecer de descrever auth**: campos `security` ausentes deixam clientes inseguros sobre quais endpoints exigem token.
- **Exemplos fora de sintonia com schema**: dev muda schema e esquece de atualizar `examples`. Use ferramenta de linting (Spectral) com regra `oas3-valid-media-example`.
- **Spec gigante de 10k linhas em um arquivo**: difícil revisar e dar merge sem conflito. Use `$ref` para arquivos externos ou ferramentas de splitting (Redocly CLI bundle).
- **Não versionar a spec junto do código**: spec em Confluence, código em Git. Diverge em semanas. Coloque `openapi.yaml` no repositório, faça PR alterando.
- **Confundir `nullable: true` (3.0) com `type: [string, "null"]` (3.1)**: usar a sintaxe errada na versão errada produz validação silenciosamente quebrada.
- **Não usar `operationId`**: clientes gerados ficam com nomes esquisitos (`postOrders` em vez de `createOrder`). Defina sempre, em snake/camel consistente.
- **Schema sem `format`**: `string` que deveria ser `email`, `uri`, `uuid`, `date-time` passa qualquer coisa. Use `format` para validação semântica.

## Links relacionados

- [[REST-Maduro]]
- [[PSRs]]
- [[Versionamento-Idempotencia]]
- [[JWT-vs-Sessoes]]
- [[Paginacao-Cursor]]
- [[Infra/CI-CD]]

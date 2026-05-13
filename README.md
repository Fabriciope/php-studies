# PHP Moderno — Guia de Estudos Aprofundados

<a href="https://fabriciopa.com.br/artigos/php-studies" target="_blank">Artigo de referência.</a>

Guia pessoal de 16 semanas para estudar PHP 8.3 / 8.4 com foco em back-end profundo: tipos, OOP moderno, padrões, DDD/Clean/Hexagonal, APIs maduras, performance e infraestrutura.

Este repositório contém **dois artefatos** que se complementam:

- `index.html` — página estática com a visão geral do roadmap, fases, exemplos-chave e método de revisão.
- `notes/` — vault Obsidian com 40 notas individuais (média 250-400 linhas) para estudo e revisão espaçada.

## Estrutura

```
.
├── index.html              ← guia visual (single-file, abrir no browser)
├── notes/                  ← vault Obsidian
│   ├── 00-Index.md         ← MOC com links para todas as notas
│   ├── 01-Roadmap.md       ← cronograma de 16 semanas
│   ├── Fundamentos/
│   ├── OOP-Moderno/
│   ├── Padroes/
│   ├── Arquitetura/
│   ├── APIs/
│   ├── Performance/
│   └── Infra/
└── README.md
```

## Como abrir o `index.html`

A página é completamente estática, sem build. Não há dependências locais — fontes vêm do Google Fonts via CDN.

**Opção 1 — abrir direto:**

```bash
# macOS
open index.html

# Linux
xdg-open index.html

# Windows
start index.html
```

**Opção 2 — servir local (recomendado se quiser permitir ancoragem por hash limpa):**

```bash
# com PHP
php -S localhost:8080

# com Python
python3 -m http.server 8080
```

Acesse `http://localhost:8080`.

A página tem dark mode embutido, sidebar fixa de navegação, oito cards detalhados (um por fase), exemplos de código PHP com highlight via CSS e seções de filosofia de estudo, exemplos-chave, recursos e método de revisão.

## Como importar `notes/` no Obsidian

1. Baixe o Obsidian em [obsidian.md](https://obsidian.md).
2. Abra o app → **Open folder as vault**.
3. Selecione a pasta `notes/` deste repositório.
4. Pronto — o vault carrega `00-Index.md` como ponto de partida (MOC).

### Recomendações de plugins

- **Graph view** (nativo) — visualizar conexões entre notas via wiki-links.
- **Dataview** — listar notas por `status` ou `fase` do frontmatter:

  ```dataview
  TABLE status, fase
  FROM "Fundamentos" OR "OOP-Moderno"
  SORT fase ASC
  ```

- **Templater** — para criar novas notas seguindo o template padrão (Conceito / Por que importa / Exemplo / Quando usar / Armadilhas / Links).

### Convenções do vault

- Frontmatter YAML em toda nota com `tags`, `fase`, `status`.
- Estados de status: `stub` → `draft` → `review` → `stable`.
- Links sempre via `[[wiki-link]]` (Obsidian resolve o caminho automaticamente).
- Cada nota segue seis seções fixas: **Conceito**, **Por que importa**, **Exemplo de código PHP**, **Quando usar / quando não usar**, **Armadilhas comuns**, **Links relacionados**.

## Como usar o guia

1. Comece pelo `index.html` para a visão de mapa.
2. Abra `notes/01-Roadmap.md` para o cronograma semanal.
3. Trabalhe **uma fase por vez**, sem pular — cada uma exige um entregável obrigatório.
4. Aplique o método de revisão de três loops descrito na página (diário, semanal, por fase).
5. Ao final da fase 8, o entregável é o portfólio.

## Cobertura

- **40 notas** distribuídas em 7 áreas (Fundamentos · OOP-Moderno · Padroes · Arquitetura · APIs · Performance · Infra)
- **8 fases** alinhadas com o cronograma
- **PHP 8.3 / 8.4** como alvo (property hooks, asymmetric visibility, readonly classes)
- Sem framework no caminho — PHP puro, padrões e infra
- Cada nota segue 6 seções fixas: Conceito, Por que importa, Exemplo de código PHP, Quando usar, Armadilhas comuns, Links relacionados
- Composer coberto em [[Fundamentos/Composer]]

## Licença

Material de estudo pessoal. Use, fork, adapte como quiser.

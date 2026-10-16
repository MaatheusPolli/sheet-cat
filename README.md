# SheetCat 🐱📊

[![CI](https://github.com/maatheuspolli/sheet-cat/actions/workflows/ci.yml/badge.svg)](https://github.com/maatheuspolli/sheet-cat/actions/workflows/ci.yml)
[![node](https://img.shields.io/badge/node-%E2%89%A522.12-green.svg)](package.json)
[![Licença: MIT pessoal + comercial](https://img.shields.io/badge/licen%C3%A7a-MIT%20pessoal%20%2B%20comercial-blue.svg)](LICENSE)

> CLI que converte planilhas de produtos (CSV/XLSX) em catálogo pronto para e-commerce, com validação linha a linha, relatório de erros e múltiplos formatos de saída.

```bash
npx github:maatheuspolli/sheet-cat convert produtos.xlsx --map vtex --out catalogo.json --report erros.csv
```

![demo do sheet-cat no terminal](docs/demo.gif)

Toda migração ou carga inicial de e-commerce começa com uma planilha do cliente: colunas com nomes inconsistentes, preço com vírgula, EAN inválido, SKU duplicado, categoria em texto livre. O SheetCat transforma essa planilha em um catálogo no formato do destino **e** diz exatamente o que está errado, linha a linha, **antes** da importação:

```
✖ linha 42, coluna "EAN": "789100" não é um EAN-13 válido (13 dígitos esperados, encontrados 6)
✖ linha 57, coluna "Cód. Produto": SKU "BOL-200" duplicado (primeira ocorrência na linha 6)
✖ linha 61, coluna "Código de Barras": "7,89100E+12" está em notação científica e perdeu dígitos; formate a coluna como texto
```

## Sumário

- [Instalação](#instalação)
- [Uso rápido](#uso-rápido)
- [Comandos](#comandos)
- [Perfis](#perfis)
- [O que é normalizado e validado](#o-que-é-normalizado-e-validado)
- [Formatos de saída](#formatos-de-saída)
- [Relatório de erros](#relatório-de-erros)
- [Exit codes e uso em CI](#exit-codes-e-uso-em-ci)
- [API programática](#api-programática)
- [Desempenho](#desempenho)
- [Arquitetura](#arquitetura)
- [Desenvolvimento](#desenvolvimento)

## Instalação

Requer **Node.js 22.12+**. O pacote não é publicado no npm; instale direto do GitHub (o build roda na instalação):

```bash
npx github:maatheuspolli/sheet-cat --help           # sem instalar
npm install -g github:maatheuspolli/sheet-cat       # ou global → comando `sheet-cat`

# ou a partir do código
git clone https://github.com/maatheuspolli/sheet-cat && cd sheet-cat
npm install && npm link
```

## Uso rápido

```bash
# 1. Veja o que tem na planilha e como as colunas serão mapeadas
sheet-cat inspect produtos.xlsx

# 2. Valide sem gravar nada
sheet-cat convert produtos.xlsx --map vtex --dry-run

# 3. Converta, com relatório das linhas rejeitadas
sheet-cat convert produtos.xlsx --map vtex --out catalogo.json --report erros.csv

# Colunas com nomes estranhos? Deixe o SheetCat sugerir o mapeamento
sheet-cat convert produtos.csv --auto-map

# ...e salve o mapeamento em um perfil reutilizável
sheet-cat init-profile cliente-x.yml --from produtos.csv --base vtex
sheet-cat convert produtos.csv --map cliente-x.yml --out catalogo.json
```

## Comandos

| Comando | O que faz |
|---------|-----------|
| `convert <planilha>` | Lê, mapeia, normaliza, valida e grava o catálogo (stdout ou `--out`). |
| `validate <planilha>` | Só valida. Exit `1` se houver erros (com `--strict`, avisos também). `--json` imprime o resumo em JSON. |
| `inspect <planilha>` | Mostra formato, encoding, delimitador, abas, colunas → campos e uma amostra (`-n`). `--json` disponível. |
| `init-profile [arquivo]` | Cria um perfil YAML; com `--from <planilha>` já sugere o mapeamento das colunas. |

### Opções principais do `convert`

| Opção | Descrição |
|-------|-----------|
| `-m, --map <perfil>` | `generic` (padrão), `vtex`, `tray` ou caminho de um `.yml` |
| `--auto-map` | mapeia colunas por similaridade e pede confirmação das ambíguas (`-y` aceita tudo) |
| `-o, --out <arquivo>` | saída (padrão: stdout, para usar em pipe) |
| `-f, --format <fmt>` | `json`, `csv`, `vtex`, `tray` (padrão: o do perfil; `.csv` no `--out` vira CSV no perfil genérico) |
| `-r, --report <arquivo>` | relatório de erros `.csv` ou `.json` |
| `--group-by [campo]` / `--no-group-by` | agrupa variações por SKU pai (ou outro campo) |
| `--dry-run` | processa tudo e não grava nenhum arquivo |
| `--strict` | qualquer erro → exit `1` e **nenhuma** saída gravada (o relatório é gravado) |
| `--force` | permite sobrescrever arquivos existentes (sem ele, o SheetCat se recusa) |
| `--sheet <aba>` | aba do XLSX por nome ou número (padrão: a primeira) |
| `--encoding`, `--delimiter` | forçam o que normalmente é detectado no CSV |
| `-q, --quiet` | sem progresso nem resumo |

Arquivos de saída são gravados de forma **atômica** (temporário + rename): uma execução interrompida ou barrada pelo `--strict` nunca deixa um catálogo pela metade nem destrói o anterior.

## Perfis

O comportamento por destino mora em YAML, não em `if` espalhado pelo código. Os perfis são validados por schema (Zod) e podem herdar uns dos outros.

| Perfil | Saída | Agrupa variações | Obrigatórios | Observações |
|--------|-------|------------------|--------------|-------------|
| [`generic`](profiles/generic.yml) | JSON canônico | não (use `--group-by`) | sku, name, price | ~120 apelidos de colunas comuns em planilhas brasileiras |
| [`vtex`](profiles/vtex.yml) | Catalog API (produto + SKUs) | sim, por `parent_sku` | sku, name, price, brand, category | aceita cabeçalhos da planilha de importação VTEX (`_IDSKU`, `_NomeProduto`...); peso em gramas |
| [`tray`](profiles/tray.yml) | API de produtos/variações | sim, por `parent_sku` | sku, name, price, ncm | `price` = preço cheio, `promotional_price` = preço "por"; peso em gramas |

Um perfil próprio:

```yaml
name: cliente-x
extends: vtex                 # herda colunas, obrigatórios e padrões
output: vtex                  # json | csv | vtex | tray
units:
  weight: g                   # unidade assumida quando a célula não diz (g, kg, mg, lb, oz)
  length: mm                  # mm, cm, m, in
required: [sku, name, price]
columns:                      # apelidos aceitos (sem diferenciar acento, caixa ou pontuação)
  sku: ["Cod. Produto", "Código Interno"]
  price: ["Preço Loja"]
defaults:                     # valor usado quando a célula está vazia
  active: true
group_by: parent_sku
```

Os apelidos do perfil filho têm prioridade; os do pai continuam aceitos. Comparação normalizada: `"Cód. Produto"`, `"COD_PRODUTO"` e `"cod produto"` são a mesma coluna.

### Auto-map

Com `--auto-map`, cada coluna é comparada com os apelidos do perfil por similaridade (tokens com abreviações — `Cod.Prod` ≈ `Código Produto` —, Levenshtein e igualdade normalizada). Sugestões com confiança baixa, ou com um segundo campo quase empatado, são marcadas como ambíguas e o SheetCat pergunta:

```
Mapeamento sugerido:
  Coluna              Campo        Confiança
  Cod.Prod            sku          95%
  Descrição do Item   description  71% ⚠ confirmar
  PREÇO VENDA         price        100%
  Qtd. Estoque        stock        71% ⚠ confirmar
  Observação Interna  (ignorada)
```

Fora de um terminal interativo (CI), o `--auto-map` exige `--yes` — nada é adivinhado em silêncio.

## O que é normalizado e validado

**Campos do modelo canônico:** `sku`, `parent_sku`, `name`, `description`, `brand`, `category`, `price`, `list_price`, `cost_price`, `stock`, `ean`, `ncm`, `weight_kg`, `height_cm`, `width_cm`, `length_cm`, `active`, `color`, `size`, `images`.

| Tipo | Exemplos de entrada | Resultado |
|------|--------------------|-----------|
| Preço | `R$ 1.234,56` · `1,234.56` · `10,5` · `1.500` | `1234.56` · `1234.56` · `10.5` · `1500` |
| Peso | `1,5 kg` · `1500g` · `850 gr` · `16 oz` · `0,3` | kg: `1.5` · `1.5` · `0.85` · `0.4536` · `0.3` (unidade do perfil) |
| Dimensão | `12,5 cm` · `125mm` · `1,2 m` · `10"` | cm: `12.5` · `12.5` · `120` · `25.4` |
| Booleano | `sim` `S` `1` `x` `ativo` / `não` `N` `0` `inativo` | `true` / `false` |
| Texto | `"  Tênis  Corrida  Pro "` | `"Tênis Corrida Pro"` (trim, espaços, invisíveis, NFC) |
| Códigos | `6404.11.00` · `789-1000-100103` | `64041100` · `7891000100103` |
| Lista | `a.jpg \| b.jpg; c.jpg` | `["a.jpg", "b.jpg", "c.jpg"]` |

Regra de separador decimal: com `.` e `,` o último é o decimal; só `,` → decimal; só `.` seguido de exatamente 3 dígitos → milhar (convenção brasileira: `1.500` = mil e quinhentos). Números vindos de células numéricas do XLSX não passam por essa heurística.

| Validação | Severidade |
|-----------|-----------|
| SKU obrigatório e **único no arquivo inteiro** (aponta a linha da primeira ocorrência) | erro |
| EAN-13 com 13 dígitos e dígito verificador correto | erro |
| EAN que o Excel converteu em notação científica (`7,89E+12`) | erro |
| NCM com 8 dígitos (máscara `0000.00.00` aceita) | erro |
| Preço, preço de lista e custo ≥ 0 | erro |
| Campos obrigatórios do perfil | erro |
| Formato inválido (preço `"dez reais"`, booleano `"talvez"`, peso `"abc"`) | erro |
| Preço de lista menor que o preço de venda | aviso |

Erros rejeitam a linha; avisos não. Todos os problemas de uma linha são acumulados (não para no primeiro).

**Leitura:** CSV/TSV com detecção de encoding (UTF-8, UTF-16 com BOM, Windows-1252/ISO-8859-1 — muito comum em planilhas brasileiras) e de delimitador (`,` `;` tab `|`); campos com quebra de linha; linhas em branco ignoradas; cabeçalho duplicado vira `Preço (2)`. XLSX pela primeira aba ou `--sheet`, com rich text, fórmulas (valor calculado) e hyperlinks.

## Formatos de saída

<details>
<summary><b>json</b> — modelo canônico (grupos com <code>variations</code>)</summary>

```json
[
  {
    "sku": "CAM-001",
    "name": "Camiseta Básica",
    "brand": "Malha & Cia",
    "variations": [
      { "sku": "CAM-001-P", "parent_sku": "CAM-001", "price": 49.9, "list_price": 69.9, "stock": 10, "color": "Azul", "size": "P", "weight_kg": 0.2 }
    ]
  }
]
```
</details>

<details>
<summary><b>vtex</b> — Catalog API: produto com SKUs</summary>

```json
[
  {
    "RefId": "TEN-100",
    "Name": "Tênis Corrida Pro",
    "BrandName": "Veloz",
    "CategoryPath": "Calçados/Tênis",
    "TaxCode": "64041100",
    "IsActive": true,
    "Skus": [
      {
        "RefId": "TEN-100-38",
        "Name": "Tênis Corrida Pro - Preto / 38",
        "Ean": "7891000200384",
        "Price": 1299.9,
        "ListPrice": 1499.9,
        "Stock": 3,
        "PackagedWeightKg": 850,
        "PackagedHeight": 12,
        "Specifications": [{ "Name": "Cor", "Value": "Preto" }, { "Name": "Tamanho", "Value": "38" }]
      }
    ]
  }
]
```

`PackagedWeightKg` sai em gramas (o padrão das lojas VTEX, apesar do nome). Marca e categoria saem por nome; a resolução de IDs acontece no envio à API (v2).
</details>

<details>
<summary><b>tray</b> — API de produtos e variações</summary>

```json
[
  {
    "Product": { "reference": "TEN-100", "name": "Tênis Corrida Pro", "ncm": "64041100", "price": 1499.9, "promotional_price": 1299.9, "weight": 850, "stock": 3, "available": 1 },
    "Variants": [
      { "Variant": { "reference": "TEN-100-38", "ean": "7891000200384", "price": 1499.9, "promotional_price": 1299.9, "Sku": [{ "type": "Cor", "value": "Preto" }, { "type": "Tamanho", "value": "38" }] } }
    ]
  }
]
```
</details>

<details>
<summary><b>csv</b> — uma linha por SKU, todas as colunas do modelo canônico</summary>

```csv
sku,parent_sku,name,description,brand,category,price,list_price,cost_price,stock,ean,ncm,weight_kg,height_cm,width_cm,length_cm,active,color,size,images
CAM-001-P,CAM-001,Camiseta Básica,,Malha & Cia,Roupas > Camisetas,49.9,69.9,,10,7891000100103,61091000,0.2,2,30,40,true,Azul,P,a.jpg|b.jpg
```
</details>

## Relatório de erros

`--report erros.csv` (ou `.json`) grava um problema por linha:

| linha | sku | campo | coluna | valor | severidade | codigo | motivo |
|------:|-----|-------|--------|-------|-----------|--------|--------|
| 5 | TEN-100-40 | ean | Código de Barras | 789100 | erro | invalid_ean | "789100" não é um EAN-13 válido (13 dígitos esperados, encontrados 6) |
| 13 | BOL-200 | sku | Cód. Produto | BOL-200 | erro | duplicate_sku | SKU "BOL-200" duplicado (primeira ocorrência na linha 6) |
| 12 | MEI-600 | list_price | Preço De | 15,00 | aviso | list_price_below_price | preço de lista (15,00) menor que o preço de venda (19,90) |

`linha` é a linha do arquivo original (1 = cabeçalho), então dá para abrir a planilha e ir direto ao ponto. `valor` é o conteúdo cru da célula, antes da normalização.

## Exit codes e uso em CI

| Código | Significado |
|-------:|-------------|
| `0` | sucesso (no `convert` sem `--strict`, linhas rejeitadas vão para o relatório) |
| `1` | erro de validação: `validate` com erros, `convert --strict` com erros, coluna obrigatória ausente |
| `2` | erro de uso: opção inválida, arquivo inexistente, perfil inválido, saída já existente sem `--force` |

Saída colorida só em terminal; limpa quando redirecionada. Respeita [`NO_COLOR`](https://no-color.org) e `FORCE_COLOR`. Mensagens e progresso vão para o **stderr**; o catálogo, para o **stdout** — dá para encadear:

```bash
sheet-cat convert produtos.csv -m vtex -q | jq '.[].Skus | length'
sheet-cat validate produtos.xlsx --map cliente-x.yml --strict --report erros.csv || exit 1
```

## API programática

Toda a lógica é exportada; a CLI é só uma casca.

```ts
import { createWriteStream } from 'node:fs';
import { openSheet, loadProfile, resolveMapping, processRows, createWriter, createErrorReport } from 'sheet-cat';

const profile = await loadProfile('vtex');
const source = await openSheet('produtos.xlsx', { sheet: 'Produtos' });
const { mapping, missingRequired } = resolveMapping(source.headers, profile);

const summary = await processRows(source.rows(), {
  profile,
  mapping,
  groupBy: profile.group_by,
  writer: createWriter('vtex', createWriteStream('catalogo.json')),
  report: createErrorReport('csv', createWriteStream('erros.csv')),
});
console.log(`${summary.valid} válidas, ${summary.rejected} rejeitadas`);
```

Peças menores também são públicas: `normalizePrice`, `normalizeWeight`, `isValidEan13`, `autoMap`, `createValidator` (aceita regras extras), `groupVariations`, `toVtexProduct`, `toTrayEntry`...

## Desempenho

Streaming de ponta a ponta: nenhuma etapa carrega o arquivo inteiro. O benchmark (`npm run bench`) gera 100 mil linhas, converte com o **binário real** e mede o pico de memória do processo pelo sistema operacional; a CI falha se passar de 300 MB.

| Cenário (100 mil linhas, 18 colunas) | Tempo | Pico RSS | Pico heap |
|---------------------------------------|------:|---------:|----------:|
| CSV (21 MB) → JSON + relatório | ~4,4 s | ~124 MB | ~41 MB |
| CSV com `--group-by` | ~4,3 s | ~191 MB | ~100 MB |
| XLSX (10 MB) → JSON + relatório | ~10,6 s | ~266 MB | ~146 MB |

O heap real retido é pequeno (o processo completa com `--max-old-space-size=48`); os picos maiores são coleta de lixo preguiçosa do V8. Agrupar variações exige guardar os produtos válidos até o fim, por isso usa mais memória. Medido em um container Linux, Node 22.

> O leitor streaming do `exceljs` tem dois bugs que o SheetCat corrige (ambos com teste de regressão):
> - corrompe caracteres acentuados que caem na fronteira entre dois pedaços do arquivo (`"não"` → `"n��o"`) — o UTF-8 é decodificado antes do parser XML;
> - o `unzipper` que ele usa sinaliza o fim do zip antes de entregar as últimas entradas, que são perdidas de forma intermitente (em arquivos com `xl/workbook.xml` no fim, cerca de metade das leituras falhava) — só o fim real do stream é repassado.

## Arquitetura

```
CLI (commander + @clack/prompts)
 └── pipeline por streaming (AsyncIterable)
      readers/     csv | xlsx → RawRow            ← únicos adapters de entrada
        ↓
      mapping/     RawRow → MappedRow             (perfil YAML ou auto-map)
        ↓
      normalize/   funções puras por tipo de campo
        ↓
      validate/    regras → Product | ValidationIssue[]
        ↓
      transform/   agrupamento de variações
        ↓
      writers/     json | csv | vtex | tray       ← únicos adapters de saída
      reporters/   relatório de erros + resumo
```

- **Erros acumulados, não exceções:** cada linha vira um produto válido ou uma lista de problemas; a CLI decide continuar ou abortar.
- **Core sem I/O:** `runPipeline`, normalizadores e validadores são testáveis sem tocar disco.
- **Perfil declarativo:** diferenças entre destinos ficam no YAML.

## Desenvolvimento

```bash
npm install
npm test               # vitest (unit, integração com snapshot, CLI em processo e via execa)
npm run coverage       # mínimo de 85% (atual: ~97% statements)
npm run typecheck
npm run build          # tsup → dist/ (ESM)
npm run bench          # benchmark de memória, 100 mil linhas
npm run dev -- inspect tests/fixtures/dirty.csv
```

- `tests/fixtures/` tem planilhas de exemplo (limpa, suja em Windows-1252, cabeçalhos acentuados/torturados, XLSX com várias abas); regenere com `npx tsx scripts/make-fixtures.ts`.
- A demo do topo é gravada executando a CLI de verdade: `npm run build && npx tsx scripts/record-demo.ts` gera `docs/demo.cast`; o GIF sai com [agg](https://github.com/asciinema/agg): `agg --font-size 16 --theme monokai --idle-time-limit 3 docs/demo.cast docs/demo.gif`.

### Fora de escopo (v1)

- Envio para a API do marketplace (v2).
- Interface gráfica.

## Status

Implementado (CLI `convert`, `validate`, `inspect` e `init-profile`, com benchmarks).

Pendências conhecidas:

- publicar no npm
- benchmark de 100 mil linhas na CI
- GIF do terminal

## Licença

Este projeto usa **licença dupla**. O texto completo está em [LICENSE](LICENSE).

- **Uso pessoal, educacional e não comercial:** livre, nos termos da licença MIT (mantenha o aviso de copyright).
- **Uso comercial:** exige licença comercial. Isso inclui usar em produto ou serviço comercial, vender acesso ou derivados, usar em atividades com fins lucrativos e implantar em empresas.

Contato para licença comercial: ma.polli@hotmail.com

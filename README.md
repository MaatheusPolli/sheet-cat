# SheetCat ðŸ±ðŸ“Š

[![CI](https://github.com/maatheuspolli/sheet-cat/actions/workflows/ci.yml/badge.svg)](https://github.com/maatheuspolli/sheet-cat/actions/workflows/ci.yml)
[![node](https://img.shields.io/badge/node-%E2%89%A522.12-green.svg)](package.json)
[![license](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

> CLI que converte planilhas de produtos (CSV/XLSX) em catÃ¡logo pronto para e-commerce, com validaÃ§Ã£o linha a linha, relatÃ³rio de erros e mÃºltiplos formatos de saÃ­da.

```bash
npx github:maatheuspolli/sheet-cat convert produtos.xlsx --map vtex --out catalogo.json --report erros.csv
```

![demo do sheet-cat no terminal](docs/demo.gif)

Toda migraÃ§Ã£o ou carga inicial de e-commerce comeÃ§a com uma planilha do cliente: colunas com nomes inconsistentes, preÃ§o com vÃ­rgula, EAN invÃ¡lido, SKU duplicado, categoria em texto livre. O SheetCat transforma essa planilha em um catÃ¡logo no formato do destino **e** diz exatamente o que estÃ¡ errado, linha a linha, **antes** da importaÃ§Ã£o:

```
âœ– linha 42, coluna "EAN": "789100" nÃ£o Ã© um EAN-13 vÃ¡lido (13 dÃ­gitos esperados, encontrados 6)
âœ– linha 57, coluna "CÃ³d. Produto": SKU "BOL-200" duplicado (primeira ocorrÃªncia na linha 6)
âœ– linha 61, coluna "CÃ³digo de Barras": "7,89100E+12" estÃ¡ em notaÃ§Ã£o cientÃ­fica e perdeu dÃ­gitos; formate a coluna como texto
```

## SumÃ¡rio

- [InstalaÃ§Ã£o](#instalaÃ§Ã£o)
- [Uso rÃ¡pido](#uso-rÃ¡pido)
- [Comandos](#comandos)
- [Perfis](#perfis)
- [O que Ã© normalizado e validado](#o-que-Ã©-normalizado-e-validado)
- [Formatos de saÃ­da](#formatos-de-saÃ­da)
- [RelatÃ³rio de erros](#relatÃ³rio-de-erros)
- [Exit codes e uso em CI](#exit-codes-e-uso-em-ci)
- [API programÃ¡tica](#api-programÃ¡tica)
- [Desempenho](#desempenho)
- [Arquitetura](#arquitetura)
- [Desenvolvimento](#desenvolvimento)

## InstalaÃ§Ã£o

Requer **Node.js 22.12+**. O pacote nÃ£o Ã© publicado no npm; instale direto do GitHub (o build roda na instalaÃ§Ã£o):

```bash
npx github:maatheuspolli/sheet-cat --help           # sem instalar
npm install -g github:maatheuspolli/sheet-cat       # ou global â†’ comando `sheet-cat`

# ou a partir do cÃ³digo
git clone https://github.com/maatheuspolli/sheet-cat && cd sheet-cat
npm install && npm link
```

## Uso rÃ¡pido

```bash
# 1. Veja o que tem na planilha e como as colunas serÃ£o mapeadas
sheet-cat inspect produtos.xlsx

# 2. Valide sem gravar nada
sheet-cat convert produtos.xlsx --map vtex --dry-run

# 3. Converta, com relatÃ³rio das linhas rejeitadas
sheet-cat convert produtos.xlsx --map vtex --out catalogo.json --report erros.csv

# Colunas com nomes estranhos? Deixe o SheetCat sugerir o mapeamento
sheet-cat convert produtos.csv --auto-map

# ...e salve o mapeamento em um perfil reutilizÃ¡vel
sheet-cat init-profile cliente-x.yml --from produtos.csv --base vtex
sheet-cat convert produtos.csv --map cliente-x.yml --out catalogo.json
```

## Comandos

| Comando | O que faz |
|---------|-----------|
| `convert <planilha>` | LÃª, mapeia, normaliza, valida e grava o catÃ¡logo (stdout ou `--out`). |
| `validate <planilha>` | SÃ³ valida. Exit `1` se houver erros (com `--strict`, avisos tambÃ©m). `--json` imprime o resumo em JSON. |
| `inspect <planilha>` | Mostra formato, encoding, delimitador, abas, colunas â†’ campos e uma amostra (`-n`). `--json` disponÃ­vel. |
| `init-profile [arquivo]` | Cria um perfil YAML; com `--from <planilha>` jÃ¡ sugere o mapeamento das colunas. |

### OpÃ§Ãµes principais do `convert`

| OpÃ§Ã£o | DescriÃ§Ã£o |
|-------|-----------|
| `-m, --map <perfil>` | `generic` (padrÃ£o), `vtex`, `tray` ou caminho de um `.yml` |
| `--auto-map` | mapeia colunas por similaridade e pede confirmaÃ§Ã£o das ambÃ­guas (`-y` aceita tudo) |
| `-o, --out <arquivo>` | saÃ­da (padrÃ£o: stdout, para usar em pipe) |
| `-f, --format <fmt>` | `json`, `csv`, `vtex`, `tray` (padrÃ£o: o do perfil; `.csv` no `--out` vira CSV no perfil genÃ©rico) |
| `-r, --report <arquivo>` | relatÃ³rio de erros `.csv` ou `.json` |
| `--group-by [campo]` / `--no-group-by` | agrupa variaÃ§Ãµes por SKU pai (ou outro campo) |
| `--dry-run` | processa tudo e nÃ£o grava nenhum arquivo |
| `--strict` | qualquer erro â†’ exit `1` e **nenhuma** saÃ­da gravada (o relatÃ³rio Ã© gravado) |
| `--force` | permite sobrescrever arquivos existentes (sem ele, o SheetCat se recusa) |
| `--sheet <aba>` | aba do XLSX por nome ou nÃºmero (padrÃ£o: a primeira) |
| `--encoding`, `--delimiter` | forÃ§am o que normalmente Ã© detectado no CSV |
| `-q, --quiet` | sem progresso nem resumo |

Arquivos de saÃ­da sÃ£o gravados de forma **atÃ´mica** (temporÃ¡rio + rename): uma execuÃ§Ã£o interrompida ou barrada pelo `--strict` nunca deixa um catÃ¡logo pela metade nem destrÃ³i o anterior.

## Perfis

O comportamento por destino mora em YAML, nÃ£o em `if` espalhado pelo cÃ³digo. Os perfis sÃ£o validados por schema (Zod) e podem herdar uns dos outros.

| Perfil | SaÃ­da | Agrupa variaÃ§Ãµes | ObrigatÃ³rios | ObservaÃ§Ãµes |
|--------|-------|------------------|--------------|-------------|
| [`generic`](profiles/generic.yml) | JSON canÃ´nico | nÃ£o (use `--group-by`) | sku, name, price | ~120 apelidos de colunas comuns em planilhas brasileiras |
| [`vtex`](profiles/vtex.yml) | Catalog API (produto + SKUs) | sim, por `parent_sku` | sku, name, price, brand, category | aceita cabeÃ§alhos da planilha de importaÃ§Ã£o VTEX (`_IDSKU`, `_NomeProduto`...); peso em gramas |
| [`tray`](profiles/tray.yml) | API de produtos/variaÃ§Ãµes | sim, por `parent_sku` | sku, name, price, ncm | `price` = preÃ§o cheio, `promotional_price` = preÃ§o "por"; peso em gramas |

Um perfil prÃ³prio:

```yaml
name: cliente-x
extends: vtex                 # herda colunas, obrigatÃ³rios e padrÃµes
output: vtex                  # json | csv | vtex | tray
units:
  weight: g                   # unidade assumida quando a cÃ©lula nÃ£o diz (g, kg, mg, lb, oz)
  length: mm                  # mm, cm, m, in
required: [sku, name, price]
columns:                      # apelidos aceitos (sem diferenciar acento, caixa ou pontuaÃ§Ã£o)
  sku: ["Cod. Produto", "CÃ³digo Interno"]
  price: ["PreÃ§o Loja"]
defaults:                     # valor usado quando a cÃ©lula estÃ¡ vazia
  active: true
group_by: parent_sku
```

Os apelidos do perfil filho tÃªm prioridade; os do pai continuam aceitos. ComparaÃ§Ã£o normalizada: `"CÃ³d. Produto"`, `"COD_PRODUTO"` e `"cod produto"` sÃ£o a mesma coluna.

### Auto-map

Com `--auto-map`, cada coluna Ã© comparada com os apelidos do perfil por similaridade (tokens com abreviaÃ§Ãµes â€” `Cod.Prod` â‰ˆ `CÃ³digo Produto` â€”, Levenshtein e igualdade normalizada). SugestÃµes com confianÃ§a baixa, ou com um segundo campo quase empatado, sÃ£o marcadas como ambÃ­guas e o SheetCat pergunta:

```
Mapeamento sugerido:
  Coluna              Campo        ConfianÃ§a
  Cod.Prod            sku          95%
  DescriÃ§Ã£o do Item   description  71% âš  confirmar
  PREÃ‡O VENDA         price        100%
  Qtd. Estoque        stock        71% âš  confirmar
  ObservaÃ§Ã£o Interna  (ignorada)
```

Fora de um terminal interativo (CI), o `--auto-map` exige `--yes` â€” nada Ã© adivinhado em silÃªncio.

## O que Ã© normalizado e validado

**Campos do modelo canÃ´nico:** `sku`, `parent_sku`, `name`, `description`, `brand`, `category`, `price`, `list_price`, `cost_price`, `stock`, `ean`, `ncm`, `weight_kg`, `height_cm`, `width_cm`, `length_cm`, `active`, `color`, `size`, `images`.

| Tipo | Exemplos de entrada | Resultado |
|------|--------------------|-----------|
| PreÃ§o | `R$ 1.234,56` Â· `1,234.56` Â· `10,5` Â· `1.500` | `1234.56` Â· `1234.56` Â· `10.5` Â· `1500` |
| Peso | `1,5 kg` Â· `1500g` Â· `850 gr` Â· `16 oz` Â· `0,3` | kg: `1.5` Â· `1.5` Â· `0.85` Â· `0.4536` Â· `0.3` (unidade do perfil) |
| DimensÃ£o | `12,5 cm` Â· `125mm` Â· `1,2 m` Â· `10"` | cm: `12.5` Â· `12.5` Â· `120` Â· `25.4` |
| Booleano | `sim` `S` `1` `x` `ativo` / `nÃ£o` `N` `0` `inativo` | `true` / `false` |
| Texto | `"  TÃªnis  CorridaÂ  Pro "` | `"TÃªnis Corrida Pro"` (trim, espaÃ§os, invisÃ­veis, NFC) |
| CÃ³digos | `6404.11.00` Â· `789-1000-100103` | `64041100` Â· `7891000100103` |
| Lista | `a.jpg \| b.jpg; c.jpg` | `["a.jpg", "b.jpg", "c.jpg"]` |

Regra de separador decimal: com `.` e `,` o Ãºltimo Ã© o decimal; sÃ³ `,` â†’ decimal; sÃ³ `.` seguido de exatamente 3 dÃ­gitos â†’ milhar (convenÃ§Ã£o brasileira: `1.500` = mil e quinhentos). NÃºmeros vindos de cÃ©lulas numÃ©ricas do XLSX nÃ£o passam por essa heurÃ­stica.

| ValidaÃ§Ã£o | Severidade |
|-----------|-----------|
| SKU obrigatÃ³rio e **Ãºnico no arquivo inteiro** (aponta a linha da primeira ocorrÃªncia) | erro |
| EAN-13 com 13 dÃ­gitos e dÃ­gito verificador correto | erro |
| EAN que o Excel converteu em notaÃ§Ã£o cientÃ­fica (`7,89E+12`) | erro |
| NCM com 8 dÃ­gitos (mÃ¡scara `0000.00.00` aceita) | erro |
| PreÃ§o, preÃ§o de lista e custo â‰¥ 0 | erro |
| Campos obrigatÃ³rios do perfil | erro |
| Formato invÃ¡lido (preÃ§o `"dez reais"`, booleano `"talvez"`, peso `"abc"`) | erro |
| PreÃ§o de lista menor que o preÃ§o de venda | aviso |

Erros rejeitam a linha; avisos nÃ£o. Todos os problemas de uma linha sÃ£o acumulados (nÃ£o para no primeiro).

**Leitura:** CSV/TSV com detecÃ§Ã£o de encoding (UTF-8, UTF-16 com BOM, Windows-1252/ISO-8859-1 â€” muito comum em planilhas brasileiras) e de delimitador (`,` `;` tab `|`); campos com quebra de linha; linhas em branco ignoradas; cabeÃ§alho duplicado vira `PreÃ§o (2)`. XLSX pela primeira aba ou `--sheet`, com rich text, fÃ³rmulas (valor calculado) e hyperlinks.

## Formatos de saÃ­da

<details>
<summary><b>json</b> â€” modelo canÃ´nico (grupos com <code>variations</code>)</summary>

```json
[
  {
    "sku": "CAM-001",
    "name": "Camiseta BÃ¡sica",
    "brand": "Malha & Cia",
    "variations": [
      { "sku": "CAM-001-P", "parent_sku": "CAM-001", "price": 49.9, "list_price": 69.9, "stock": 10, "color": "Azul", "size": "P", "weight_kg": 0.2 }
    ]
  }
]
```
</details>

<details>
<summary><b>vtex</b> â€” Catalog API: produto com SKUs</summary>

```json
[
  {
    "RefId": "TEN-100",
    "Name": "TÃªnis Corrida Pro",
    "BrandName": "Veloz",
    "CategoryPath": "CalÃ§ados/TÃªnis",
    "TaxCode": "64041100",
    "IsActive": true,
    "Skus": [
      {
        "RefId": "TEN-100-38",
        "Name": "TÃªnis Corrida Pro - Preto / 38",
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

`PackagedWeightKg` sai em gramas (o padrÃ£o das lojas VTEX, apesar do nome). Marca e categoria saem por nome; a resoluÃ§Ã£o de IDs acontece no envio Ã  API (v2).
</details>

<details>
<summary><b>tray</b> â€” API de produtos e variaÃ§Ãµes</summary>

```json
[
  {
    "Product": { "reference": "TEN-100", "name": "TÃªnis Corrida Pro", "ncm": "64041100", "price": 1499.9, "promotional_price": 1299.9, "weight": 850, "stock": 3, "available": 1 },
    "Variants": [
      { "Variant": { "reference": "TEN-100-38", "ean": "7891000200384", "price": 1499.9, "promotional_price": 1299.9, "Sku": [{ "type": "Cor", "value": "Preto" }, { "type": "Tamanho", "value": "38" }] } }
    ]
  }
]
```
</details>

<details>
<summary><b>csv</b> â€” uma linha por SKU, todas as colunas do modelo canÃ´nico</summary>

```csv
sku,parent_sku,name,description,brand,category,price,list_price,cost_price,stock,ean,ncm,weight_kg,height_cm,width_cm,length_cm,active,color,size,images
CAM-001-P,CAM-001,Camiseta BÃ¡sica,,Malha & Cia,Roupas > Camisetas,49.9,69.9,,10,7891000100103,61091000,0.2,2,30,40,true,Azul,P,a.jpg|b.jpg
```
</details>

## RelatÃ³rio de erros

`--report erros.csv` (ou `.json`) grava um problema por linha:

| linha | sku | campo | coluna | valor | severidade | codigo | motivo |
|------:|-----|-------|--------|-------|-----------|--------|--------|
| 5 | TEN-100-40 | ean | CÃ³digo de Barras | 789100 | erro | invalid_ean | "789100" nÃ£o Ã© um EAN-13 vÃ¡lido (13 dÃ­gitos esperados, encontrados 6) |
| 13 | BOL-200 | sku | CÃ³d. Produto | BOL-200 | erro | duplicate_sku | SKU "BOL-200" duplicado (primeira ocorrÃªncia na linha 6) |
| 12 | MEI-600 | list_price | PreÃ§o De | 15,00 | aviso | list_price_below_price | preÃ§o de lista (15,00) menor que o preÃ§o de venda (19,90) |

`linha` Ã© a linha do arquivo original (1 = cabeÃ§alho), entÃ£o dÃ¡ para abrir a planilha e ir direto ao ponto. `valor` Ã© o conteÃºdo cru da cÃ©lula, antes da normalizaÃ§Ã£o.

## Exit codes e uso em CI

| CÃ³digo | Significado |
|-------:|-------------|
| `0` | sucesso (no `convert` sem `--strict`, linhas rejeitadas vÃ£o para o relatÃ³rio) |
| `1` | erro de validaÃ§Ã£o: `validate` com erros, `convert --strict` com erros, coluna obrigatÃ³ria ausente |
| `2` | erro de uso: opÃ§Ã£o invÃ¡lida, arquivo inexistente, perfil invÃ¡lido, saÃ­da jÃ¡ existente sem `--force` |

SaÃ­da colorida sÃ³ em terminal; limpa quando redirecionada. Respeita [`NO_COLOR`](https://no-color.org) e `FORCE_COLOR`. Mensagens e progresso vÃ£o para o **stderr**; o catÃ¡logo, para o **stdout** â€” dÃ¡ para encadear:

```bash
sheet-cat convert produtos.csv -m vtex -q | jq '.[].Skus | length'
sheet-cat validate produtos.xlsx --map cliente-x.yml --strict --report erros.csv || exit 1
```

## API programÃ¡tica

Toda a lÃ³gica Ã© exportada; a CLI Ã© sÃ³ uma casca.

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
console.log(`${summary.valid} vÃ¡lidas, ${summary.rejected} rejeitadas`);
```

PeÃ§as menores tambÃ©m sÃ£o pÃºblicas: `normalizePrice`, `normalizeWeight`, `isValidEan13`, `autoMap`, `createValidator` (aceita regras extras), `groupVariations`, `toVtexProduct`, `toTrayEntry`...

## Desempenho

Streaming de ponta a ponta: nenhuma etapa carrega o arquivo inteiro. O benchmark (`npm run bench`) gera 100 mil linhas, converte com o **binÃ¡rio real** e mede o pico de memÃ³ria do processo pelo sistema operacional; a CI falha se passar de 300 MB.

| CenÃ¡rio (100 mil linhas, 18 colunas) | Tempo | Pico RSS | Pico heap |
|---------------------------------------|------:|---------:|----------:|
| CSV (21 MB) â†’ JSON + relatÃ³rio | ~4,4 s | ~124 MB | ~41 MB |
| CSV com `--group-by` | ~4,3 s | ~191 MB | ~100 MB |
| XLSX (10 MB) â†’ JSON + relatÃ³rio | ~10,6 s | ~266 MB | ~146 MB |

O heap real retido Ã© pequeno (o processo completa com `--max-old-space-size=48`); os picos maiores sÃ£o coleta de lixo preguiÃ§osa do V8. Agrupar variaÃ§Ãµes exige guardar os produtos vÃ¡lidos atÃ© o fim, por isso usa mais memÃ³ria. Medido em um container Linux, Node 22.

> O leitor streaming do `exceljs` tem dois bugs que o SheetCat corrige (ambos com teste de regressÃ£o):
> - corrompe caracteres acentuados que caem na fronteira entre dois pedaÃ§os do arquivo (`"nÃ£o"` â†’ `"nï¿½ï¿½o"`) â€” o UTF-8 Ã© decodificado antes do parser XML;
> - o `unzipper` que ele usa sinaliza o fim do zip antes de entregar as Ãºltimas entradas, que sÃ£o perdidas de forma intermitente (em arquivos com `xl/workbook.xml` no fim, cerca de metade das leituras falhava) â€” sÃ³ o fim real do stream Ã© repassado.

## Arquitetura

```
CLI (commander + @clack/prompts)
 â””â”€â”€ pipeline por streaming (AsyncIterable)
      readers/     csv | xlsx â†’ RawRow            â† Ãºnicos adapters de entrada
        â†“
      mapping/     RawRow â†’ MappedRow             (perfil YAML ou auto-map)
        â†“
      normalize/   funÃ§Ãµes puras por tipo de campo
        â†“
      validate/    regras â†’ Product | ValidationIssue[]
        â†“
      transform/   agrupamento de variaÃ§Ãµes
        â†“
      writers/     json | csv | vtex | tray       â† Ãºnicos adapters de saÃ­da
      reporters/   relatÃ³rio de erros + resumo
```

- **Erros acumulados, nÃ£o exceÃ§Ãµes:** cada linha vira um produto vÃ¡lido ou uma lista de problemas; a CLI decide continuar ou abortar.
- **Core sem I/O:** `runPipeline`, normalizadores e validadores sÃ£o testÃ¡veis sem tocar disco.
- **Perfil declarativo:** diferenÃ§as entre destinos ficam no YAML.

## Desenvolvimento

```bash
npm install
npm test               # vitest (unit, integraÃ§Ã£o com snapshot, CLI em processo e via execa)
npm run coverage       # mÃ­nimo de 85% (atual: ~97% statements)
npm run typecheck
npm run build          # tsup â†’ dist/ (ESM)
npm run bench          # benchmark de memÃ³ria, 100 mil linhas
npm run dev -- inspect tests/fixtures/dirty.csv
```

- `tests/fixtures/` tem planilhas de exemplo (limpa, suja em Windows-1252, cabeÃ§alhos acentuados/torturados, XLSX com vÃ¡rias abas); regenere com `npx tsx scripts/make-fixtures.ts`.
- A demo do topo Ã© gravada executando a CLI de verdade: `npm run build && npx tsx scripts/record-demo.ts` gera `docs/demo.cast`; o GIF sai com [agg](https://github.com/asciinema/agg): `agg --font-size 16 --theme monokai --idle-time-limit 3 docs/demo.cast docs/demo.gif`.

### Fora de escopo (v1)

- Envio para a API do marketplace (v2).
- Interface grÃ¡fica.

## LicenÃ§a

[MIT](LICENSE) Â© 2026 Matheus Gasparotto Polli


## Licença

### Uso Pessoal
Este projeto é licenciado sob a **Licença MIT** para uso pessoal, educacional e não comercial.

### Uso Comercial
Se você deseja usar este projeto comercialmente, deve obter uma licença comercial do autor.

**Uso comercial inclui:**
- Incorporar em um produto comercial
- Oferecer como serviço
- Usar em atividades com fins lucrativos
- Implantação empresarial

**Para obter uma licença comercial:**
Email: ma.polli@hotmail.com

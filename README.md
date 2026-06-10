# Taxa SELIC e Tesouro Direto — Dados Públicos Integrados

> Trabalho Prático — Introdução a Bancos de Dados (IBD)
> Universidade Federal de Minas Gerais — DCC/UFMG — 1º Semestre de 2026

**Autores:** Davi Ladeira Balsamão · Enzo de Freitas Alencar · Fernando Thales Santana Lopes · Otávio Fagundes Rodrigues de Oliveira · Pedro Puliti Dias

---

## Sobre este repositório

Este repositório contém os dados organizados e documentação referentes à análise integrada da **taxa SELIC diária** (Banco Central do Brasil) e dos **preços e taxas do Tesouro Direto** (Tesouro Nacional), cobrindo o período de janeiro de 2021 a dezembro de 2025.

---

## Fontes de dados

| # | Fonte | Órgão produtor | URL | Licença |
|---|-------|---------------|-----|---------|
| 1 | Taxa SELIC Diária — Série SGS nº 11 | Banco Central do Brasil (BACEN) | https://dadosabertos.bcb.gov.br/dataset/11-taxa-de-juros---selic | Dados públicos — LAI (Lei nº 12.527/2011) |
| 2 | Histórico de Preços e Taxas do Tesouro Direto | Tesouro Nacional | https://www.tesourodireto.com.br/titulos/historico-de-precos-e-taxas.htm | Dados públicos — LAI (Lei nº 12.527/2011) |

### Processo de aquisição

Os arquivos foram baixados diretamente dos portais oficiais em formato CSV (separador `;`, decimais com vírgula). O script `scripts/tratar_dados.py` realizou as seguintes transformações:

- Conversão das datas de `DD/MM/AAAA` para ISO 8601 (`AAAA-MM-DD`)
- Substituição de vírgulas por pontos nos campos decimais
- Geração dos arquivos `selic_limpo.csv` e `tesouro_absoluto.csv`

**Data de obtenção:** 2026
**Cobertura temporal:** 04/01/2021 a 31/12/2025
**Total de registros:** 1.256 (SELIC) + 52.706 (Tesouro Direto) = **53.962 registros**

---

## Dicionário de dados

### Tabela `selic_diaria`

| Coluna | Tipo SQL | Chave | Descrição | Valores esperados |
|--------|----------|-------|-----------|-------------------|
| `data` | `DATE` | PK | Data útil de referência (ISO 8601) | 2021-01-04 a 2025-12-31 |
| `valor` | `NUMERIC(10,6)` | — | Taxa SELIC efetiva diária (% a.a.) | 2,0 a 14,75 no período |

### Tabela `tesouro_direto`

| Coluna | Tipo SQL | Chave | Descrição | Valores esperados |
|--------|----------|-------|-----------|-------------------|
| `tipo_titulo` | `VARCHAR(80)` | PK | Nome do título (ex.: Tesouro IPCA+ 2029) | Ver lista abaixo |
| `data_vencimento` | `DATE` | PK | Data de vencimento do título | Datas futuras a partir de 2024 |
| `data_base` | `DATE` | PK + FK | Data útil de cotação — FK para `selic_diaria.data` | 2021-01-04 a 2025-12-30 |
| `taxa_compra_manha` | `NUMERIC(8,4)` | — | Taxa de compra divulgada pela manhã (% a.a.) | Pode ser NULL¹ |
| `taxa_venda_manha` | `NUMERIC(8,4)` | — | Taxa de venda divulgada pela manhã (% a.a.) | 2,0 a 15,0 aprox. |
| `pu_compra_manha` | `NUMERIC(12,6)` | — | Preço unitário de compra pela manhã (R$) | Pode ser NULL¹ |
| `pu_venda_manha` | `NUMERIC(12,6)` | — | Preço unitário de venda pela manhã (R$) | > 0 |
| `pu_base_manha` | `NUMERIC(12,6)` | — | Preço unitário base pela manhã (R$) | > 0 |

> ¹ Campos de compra podem ser `NULL` quando o Tesouro suspendeu a recompra do título naquele dia — comportamento esperado, não erro de dados.

### Tipos de título presentes no conjunto

| Categoria | Exemplos de tipo_titulo |
|-----------|------------------------|
| Indexado à SELIC | Tesouro Selic 2026, Tesouro Selic 2027, Tesouro Selic 2029 |
| Prefixado | Tesouro Prefixado 2026, Tesouro Prefixado 2027, Tesouro Prefixado com Juros Semestrais 2033 |
| Indexado ao IPCA | Tesouro IPCA+ 2029, Tesouro IPCA+ 2035, Tesouro IPCA+ com Juros Semestrais 2040 |
| Renda+ / Educa+ | Tesouro Renda+ 2030, Tesouro Educa+ 2029 |

### Metadados gerais

| Atributo | SELIC Diária | Tesouro Direto |
|----------|-------------|----------------|
| Órgão produtor | Banco Central do Brasil | Tesouro Nacional |
| Data de obtenção | Janeiro de 2026 | Janeiro de 2026 |
| Cobertura temporal | 04/01/2021 – 31/12/2025 | 04/01/2021 – 30/12/2025 |
| Periodicidade | Diária (dias úteis) | Diária (dias úteis) |
| Cobertura geográfica | Brasil | Brasil |
| Formato original | CSV `;` — decimal `,` | CSV `;` — decimal `,` |
| Formato neste repositório | CSV `;` — decimal `.` | CSV `;` — decimal `.` |
| Licença | Dados públicos — LAI | Dados públicos — LAI |

---

*UFMG / DCC — Introdução a Bancos de Dados — 1º Semestre de 2026*

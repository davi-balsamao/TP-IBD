# Resumo Geral do Trabalho — TP IBD

> **Tema:** Análise integrada da Taxa SELIC diária (BACEN) e dos preços/taxas do
> Tesouro Direto (Tesouro Nacional), no PostgreSQL.
> **Disciplina:** Introdução a Bancos de Dados — DCC/UFMG — 2026/1.
> **Autores:** Davi Ladeira Balsamão, Enzo de Freitas Alencar, Fernando Thales Santana Lopes,
> Otávio Fagundes Rodrigues de Oliveira, Pedro Puliti Dias.
> Documento de apoio para a apresentação — pensado para responder perguntas do professor.
> Fontes: `README.md`, `docs/Relatorio_Parte1_IBD.pdf`, `docs/Relatorio_Parte2.md`,
> `docs/analise_critica.md`.

---

## 1. Visão geral em uma frase

Modelamos e carregamos no PostgreSQL duas fontes públicas — SELIC diária e Tesouro Direto
(2021–2025) — e, com 20 consultas SQL, mostramos que **quando a SELIC sobe, o prêmio dos
títulos IPCA+ aumenta e o preço dos títulos Prefixados cai** (efeito gangorra).

## 2. As fontes de dados

| Fonte | Órgão | Registros | Cobertura | Granularidade |
|---|---|---:|---|---|
| Taxa SELIC Diária (série SGS nº 11) | BACEN | 1.256 | 04/01/2021 – 31/12/2025 | Taxa efetiva **diária** (% ao dia), dias úteis |
| Preços e Taxas do Tesouro Direto | Tesouro Nacional | 52.706 | 04/01/2021 – 30/12/2025 | Cotação da manhã, dias úteis |

- **Total integrado:** 53.962 registros.
- Licença: dados públicos (LAI — Lei nº 12.527/2011).
- Aquisição: baixados em CSV (`;`, decimal `,`); script `tratar_dados.py` converteu datas para
  ISO 8601 e decimais para ponto, gerando `selic_limpo.csv` e `tesouro_absoluto.csv`.

> ⚠️ **Ponto sensível (saiba explicar):** a coluna `selic_diaria.valor` é a taxa **diária**
> (% ao dia), valores ~0,0075 a 0,0551 — **não** é a SELIC anualizada do Copom. Para anualizar:
> `(1 + taxa_diária)^252 − 1`. Referência: 0,0551%/dia ≈ 15% a.a.; 0,0075%/dia ≈ 2% a.a.

## 3. Modelagem (Parte 1)

Duas tabelas no PostgreSQL (`ibd_tp`):

**`selic_diaria`**
- `data` `DATE` — **PK**
- `valor` `NUMERIC(10,6)` — taxa SELIC efetiva diária (% ao dia)

**`tesouro_direto`**
- `tipo_titulo` `VARCHAR(80)` — **PK**
- `data_vencimento` `DATE` — **PK**
- `data_base` `DATE` — **PK + FK** → `selic_diaria.data` (data da cotação)
- `taxa_compra_manha` / `taxa_venda_manha` `NUMERIC(8,4)` — taxas (% a.a.)
- `pu_compra_manha` / `pu_venda_manha` / `pu_base_manha` `NUMERIC(12,6)` — Preços Unitários (R$)

- **Chave primária composta:** `(tipo_titulo, data_vencimento, data_base)` — garante unicidade.
- **Junção** entre as tabelas: `tesouro_direto.data_base = selic_diaria.data`.
- 8 famílias de título: Selic, Prefixado, IPCA+, IGPM+, Renda+, Educa+ e variantes "com Juros Semestrais".
- Campos de **compra** podem ser `NULL` quando o Tesouro suspende a recompra — comportamento esperado, não erro.

## 4. Metodologia (Parte 2): 20 consultas em 5 etapas

1. **Caracterização** (5 consultas) — volume, período, tipos de título, perfil da SELIC.
2. **Definição dos objetivos** (2 questões de pesquisa).
3. **Preparação** (5 consultas) — nulos, duplicatas, domínio, cronologia, atributos derivados.
4. **Análise descritiva** (5 consultas) — estatísticas de taxas e PU, SELIC por ano, volatilidade.
5. **Análise por objetivos** (5 consultas) — cruzamento SELIC × Tesouro para responder às perguntas.

## 5. Principais achados (etapa por etapa)

### Caracterização
- 53.962 registros, 2021–2025.
- Distribuição por família: **IPCA+ = 16.656**; **Prefixado = 12.875**; Educa+ = 10.369;
  Renda+ = 5.832; Selic = 5.668; IGPM+ = 1.306 (residual). Oferta dominada por IPCA+ e Educa+.
- SELIC diária: mín 0,007469; máx 0,055131; média ~0,04 (≈ ~10,6% a.a.).
- Maior taxa de venda: **16,33%** (Tesouro Prefixado).

### Preparação / qualidade
- **0 duplicatas** (chave primária íntegra).
- **0 nulos** nas colunas de **venda** (`taxa_venda_manha`, `pu_venda_manha`).
- **Cronologia consistente** (nenhum vencimento ≤ data de cotação).
- Atributos derivados criados: `ano_cotacao`, `mes_cotacao`, `dias_para_vencer`.

### Descritiva
- Prefixados: maiores taxas nominais médias (~11,8%). IPCA+: entrega taxa **real**.
- Volatilidade do PU cresce com o prazo: Prefixado venc. 2022 → amplitude R$ 35,59;
  venc. 2026 → R$ 393,19 (títulos longos têm duration maior).

### Objetivo 1 — Prêmio do IPCA+ vs. cenário da SELIC ✅ confirmado
| Cenário | SELIC diária média | Taxa média IPCA+ |
|---|---:|---:|
| Selic Alta (≥ 0,05) | 0,0526 | **6,88%** |
| Selic Intermediária | 0,0420 | 6,19% |
| Selic Baixa (≤ 0,02) | 0,0129 | **3,98%** |

O prêmio real médio do IPCA+ **quase dobra** (3,98% → 6,88%), com picos de **10,49%** em
dez/2025 (auge do aperto). O Tesouro paga mais para atrair o investidor em ciclos de juros altos.

### Objetivo 2 — Efeito gangorra no Prefixado ✅ confirmado
| Ano | SELIC diária máx | PU médio Prefixado |
|---|---:|---:|
| 2021 | 0,0347 | **933,70** |
| 2022 | 0,0508 | 848,09 |
| 2023 | 0,0508 | 873,87 |
| 2024 | 0,0455 | 864,80 |
| 2025 | 0,0551 | **773,10** |

Quando a SELIC sobe, o PU cai: de **R$ 933,70 (2021)** para **R$ 773,10 (2025)**, com mínima
histórica de **R$ 381,26**. O efeito é mais forte nos vencimentos longos.

## 6. Análise crítica (limitações das fontes)

**SELIC Diária (BACEN):**
- Cobre só dias úteis (sem fins de semana/feriados) — cuidado em retornos acumulados.
- Publicada com 1 dia útil de atraso; sem granularidade intradiária; não reflete expectativas futuras.
- É a série **diária** (SGS 11), distinta da SELIC oficial anualizada do Copom (base 252 dias úteis).

**Tesouro Direto (Tesouro Nacional):**
- Apenas cotações da manhã — sem análise intradiária.
- **Viés de sobrevivência:** títulos descontinuados antes de 2021 não aparecem.
- `tipo_titulo` mistura categoria, produto e vencimento — exige normalização.
- **Sem volume negociado** — impossível medir liquidez (por isso só há 2 objetivos, não 3).

## 7. Pontos de atenção / honestidade metodológica

Saber citar estes pontos demonstra rigor (estão documentados em `Relatorio_Parte2.md`, seção 6):

1. **Mistura de subtipos de Prefixado:** consultas com `LIKE 'Tesouro Prefixado%'` capturam tanto
   o "Tesouro Prefixado" (zero-cupom) quanto o "...com Juros Semestrais" — o PU médio mistura os dois.
2. **Valores anômalos no Tesouro:** PU = R$ 0,00 no IGPM+ e taxas negativas (IGPM+ −3,28%;
   Selic −0,04%). A verificação de domínio cobriu só a SELIC, não os preços/taxas do Tesouro.
3. **Consulta 20 redundante:** repete a 14 e faz um JOIN inútil com `selic_diaria`.
4. **Verificação de nulos parcial:** testou só colunas de venda; os nulos previstos estão nas de compra.

## 8. Possíveis perguntas do professor (FAQ)

**Por que a SELIC aparece com valores ~0,04 e não ~13%?**
Porque é a série **diária** (SGS 11), em % ao dia. Para anualizar: `(1+valor)^252 − 1`.

**O que é o PU (Preço Unitário)?**
É o preço, em reais, de uma unidade do título numa data. Sobe e desce com os juros de mercado.

**Por que o preço do Prefixado cai quando a SELIC sobe?**
O Prefixado tem taxa fixa. Se a SELIC (juros de mercado) sobe, títulos novos pagam mais; para o
título antigo competir, seu preço precisa cair — é a marcação a mercado (efeito gangorra). Mais
intenso em vencimentos longos (maior duration).

**Por que o IPCA+ paga mais em juros altos?**
Em ciclos de aperto, o Tesouro eleva o prêmio real para atrair investidores — vimos o prêmio
médio quase dobrar (3,98% → 6,88%).

**Como as duas tabelas se relacionam?**
Por data: `tesouro_direto.data_base` é FK para `selic_diaria.data`. Os JOINs casam cada cotação
do Tesouro com a SELIC do mesmo dia.

**Qual é a chave primária do Tesouro? Por que composta?**
`(tipo_titulo, data_vencimento, data_base)` — um mesmo tipo/vencimento aparece uma vez por dia de
cotação; a tripla identifica unicamente cada linha. Confirmado: 0 duplicatas.

**Tem dados faltando (nulos)?**
Nas colunas de **venda**, não — 0 nulos. Nas de **compra**, pode haver NULL quando o Tesouro
suspende a recompra (esperado). Não verificamos exaustivamente as colunas de compra.

**Por que só 2 objetivos e não análise de liquidez?**
A fonte do Tesouro não traz **volume negociado**, então não há como medir liquidez.

**E os valores negativos / PU zerado?**
São anomalias de domínio no Tesouro (IGPM+ e Selic), tratadas como limitação na análise crítica —
não como resultado. A verificação de domínio sistemática cobriu a SELIC.

**Por que PostgreSQL?**
SGBD relacional robusto, gratuito, com bom suporte a tipos `DATE`/`NUMERIC`, chaves compostas e
funções de agregação/data usadas nas consultas (`EXTRACT`, `AVG`, `GROUP BY`).

## 9. Entregáveis e publicação

- Dados tratados: `dados/selic_limpo.csv`, `dados/tesouro_absoluto.csv`.
- Modelo ER: `metadados/ModeloER.png`.
- Relatórios: `docs/Relatorio_Parte1_IBD.pdf`, `docs/Relatorio_Parte2.md`, `docs/analise_critica.md`.
- Resultados das consultas: `docs/consultas-resultados/consulta_1..20` (CSV).
- Apresentação: `docs/Apresentacao_TP-IBD.pptx`.
- Repositório: `github.com/davi-balsamao/TP-IBD`.

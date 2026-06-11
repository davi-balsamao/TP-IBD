# Trabalho Prático — Parte 2: Análise Exploratória (SQL)
**Taxa SELIC e Tesouro Direto — Análise no PostgreSQL**

> Documento consolidado da Parte 2. Reúne, para cada etapa exigida pelo enunciado,
> a consulta SQL executada, o resultado obtido e a interpretação.
> Banco: PostgreSQL (`ibd_tp`), tabelas `selic_diaria` e `tesouro_direto` (ver Parte 1).
>
> **Observação metodológica importante (ler antes de tudo):** a coluna `valor` de
> `selic_diaria` vem da série SGS nº 11 do BACEN, que é a **taxa SELIC efetiva
> diária (% ao dia)** — e **não** a taxa anualizada (% a.a.). Por isso os valores
> aparecem na casa de `0,007` a `0,055`. Sempre que este relatório fala em "SELIC",
> trata-se da taxa **diária**. (Conversão de referência: `0,0551%/dia ≈ 15% a.a.`;
> `0,0075%/dia ≈ 2% a.a.`.) O dicionário da Parte 1 descreve essa coluna como taxa
> diária; o rótulo correto da unidade é **"% ao dia"**.

---

## Índice
1. [Caracterização inicial dos dados](#1-caracterização-inicial-dos-dados) — 5 consultas
2. [Definição dos objetivos](#2-definição-dos-objetivos) — 2 questões de pesquisa
3. [Preparação dos dados](#3-preparação-dos-dados) — 5 consultas
4. [Análise descritiva](#4-análise-descritiva-dos-dados) — 5 consultas
5. [Análise orientada pelos objetivos](#5-análise-orientada-pelos-objetivos) — 5 consultas
6. [Síntese dos achados](#6-síntese-dos-achados)
7. [Análise crítica das fontes de dados](#7-análise-crítica-das-fontes-de-dados)
8. [Mapa de arquivos](#8-mapa-de-arquivos)

---

## 1. Caracterização inicial dos dados

Objetivo da etapa: compreender estrutura, abrangência e comportamento geral dos dados
antes de qualquer transformação.

### Consulta 1 — Quantificação total de registros
```sql
SELECT
    (SELECT COUNT(*) FROM selic_diaria)   AS total_registros_selic,
    (SELECT COUNT(*) FROM tesouro_direto) AS total_registros_tesouro;
```
**Resultado:** `selic_diaria = 1.256` · `tesouro_direto = 52.706`.
Confirma o volume descrito na Parte 1.

### Consulta 2 — Caracterização da dimensão temporal
```sql
SELECT
    MIN(data_base) AS primeira_data_cotacao,
    MAX(data_base) AS ultima_data_cotacao
FROM tesouro_direto;
```
**Resultado:** `2021-01-04` a `2025-12-30` (5 anos de cobertura, dias úteis).

### Consulta 3 — Distribuição de cotações por tipo de título
```sql
SELECT
    tipo_titulo,
    COUNT(*) AS quantidade_de_cotacoes
FROM tesouro_direto
GROUP BY tipo_titulo
ORDER BY quantidade_de_cotacoes DESC;
```
**Resultado:**

| tipo_titulo | cotações |
|---|---:|
| Tesouro IPCA+ com Juros Semestrais | 10.816 |
| Tesouro Educa+ | 10.369 |
| Tesouro Prefixado com Juros Semestrais | 6.670 |
| Tesouro Prefixado | 6.205 |
| Tesouro IPCA+ | 5.840 |
| Tesouro Renda+ Aposentadoria Extra | 5.832 |
| Tesouro Selic | 5.668 |
| Tesouro IGPM+ com Juros Semestrais | 1.306 |
| **Total** | **52.706** |

Agrupando por família: **IPCA+** (IPCA+ e IPCA+ c/ Juros) = **16.656**;
**Prefixado** (Prefixado e Prefixado c/ Juros) = **12.875**; **Educa+** = 10.369;
**Renda+** = 5.832; **Selic** = 5.668; **IGPM+** = 1.306. A oferta é dominada pelos
indexados à inflação (IPCA+) e pelo título temático Educa+; o IGPM+ é residual.

### Consulta 4 — Perfil de valores da Taxa SELIC (diária)
```sql
SELECT
    MIN(valor)            AS selic_minima_periodo,
    MAX(valor)            AS selic_maxima_periodo,
    ROUND(AVG(valor), 2)  AS selic_media_periodo
FROM selic_diaria;
```
**Resultado:** mínima `0,007469` · máxima `0,055131` · média `0,04` (taxa **diária**).
Em termos anualizados aproximados: de **~2% a.a.** (piso de 2021) a **~15% a.a.**
(pico de 2025), média em torno de **~10,6% a.a.** — um período de forte aperto
monetário ao longo do recorte.

### Consulta 5 — Maior taxa de venda por tipo de título
```sql
SELECT
    tipo_titulo,
    MAX(taxa_venda_manha) AS maior_taxa_venda_registrada
FROM tesouro_direto
WHERE taxa_venda_manha IS NOT NULL
GROUP BY tipo_titulo
ORDER BY maior_taxa_venda_registrada DESC;
```
**Resultado (topo):** Prefixado **16,33%**; Prefixado c/ Juros **16,31%**;
IPCA+ e IPCA+ c/ Juros **10,49%**; Educa+ 8,47%; IGPM+ 8,23%; Renda+ 7,80%;
Selic 0,36%. Os prefixados "travaram" os maiores prêmios nominais; o IPCA+
ofereceu os maiores juros **reais** (acima da inflação).

---

## 2. Definição dos objetivos

Duas questões de pesquisa orientam as etapas seguintes:

- **Objetivo 1 — O "preço" de proteger o poder de compra.** Em ciclos de alta da
  SELIC (Banco Central subindo juros para conter a inflação), o Tesouro passa a
  oferecer prêmios (taxas reais) maiores nos títulos **Tesouro IPCA+** para atrair
  o investidor?
- **Objetivo 2 — O "efeito gangorra" na prática.** Como os ciclos de alta e queda
  da SELIC afetam o **Preço Unitário (PU)** dos títulos **Tesouro Prefixado**?

---

## 3. Preparação dos dados

Objetivo da etapa: validar qualidade/consistência e criar atributos derivados.

### Consulta 6 — Verificação de valores nulos
```sql
SELECT
    tipo_titulo,
    COUNT(*) AS total_registros,
    SUM(CASE WHEN taxa_venda_manha IS NULL THEN 1 ELSE 0 END) AS qtd_taxa_nula,
    SUM(CASE WHEN pu_venda_manha   IS NULL THEN 1 ELSE 0 END) AS qtd_pu_nulo
FROM tesouro_direto
GROUP BY tipo_titulo
ORDER BY qtd_taxa_nula DESC;
```
**Resultado:** **0 nulos** em `taxa_venda_manha` e `pu_venda_manha` para **todos** os
títulos. As colunas de **venda** estão completas no período.

> ⚠️ **Ressalva:** esta consulta verifica apenas as colunas de _venda_. A limitação
> registrada na Parte 1 (nulos por suspensão de negociação) refere-se às colunas de
> **compra** (`taxa_compra_manha`/`pu_compra_manha`), que não foram testadas aqui.
> Portanto, **não se pode afirmar** que existem lacunas nos dados com base apenas
> nesta consulta — pelo contrário, nas colunas de venda não há nulos.

### Consulta 7 — Identificação de registros duplicados
```sql
SELECT
    tipo_titulo,
    data_vencimento,
    data_base,
    COUNT(*) AS qtd_repeticoes
FROM tesouro_direto
GROUP BY tipo_titulo, data_vencimento, data_base
HAVING COUNT(*) > 1;
```
**Resultado:** **vazio** (0 linhas) — a chave primária composta
`(tipo_titulo, data_vencimento, data_base)` garantiu unicidade; não há duplicatas.

### Consulta 8 — Valores fora do domínio (SELIC)
```sql
SELECT *
FROM selic_diaria
WHERE valor < 0 OR valor > 100;
```
**Resultado:** **vazio** — nenhuma SELIC negativa.

> ⚠️ **Ressalva:** o limite superior `> 100` foi pensado para taxa **anual** (% a.a.),
> mas a coluna armazena taxa **diária** (máx. observada ≈ 0,055). O teste passa, mas
> o domínio efetivo deveria ser bem mais estreito (p. ex. `valor > 1`). Além disso,
> **o domínio dos preços/taxas do Tesouro não foi verificado** — ver os valores
> anômalos apontados na seção 6 (PU = 0,00 e taxas negativas no IGPM+/Selic).

### Consulta 9 — Consistência cronológica
```sql
SELECT
    tipo_titulo,
    data_base,
    data_vencimento
FROM tesouro_direto
WHERE data_vencimento <= data_base;
```
**Resultado:** **vazio** — nenhum título com vencimento anterior/igual à data de
cotação. Cronologia consistente.

### Consulta 10 — Criação de atributos derivados
```sql
SELECT
    tipo_titulo,
    data_base,
    EXTRACT(YEAR  FROM data_base)   AS ano_cotacao,
    EXTRACT(MONTH FROM data_base)   AS mes_cotacao,
    data_vencimento,
    (data_vencimento - data_base)   AS dias_para_vencer,
    pu_venda_manha
FROM tesouro_direto
WHERE pu_venda_manha IS NOT NULL
LIMIT 20;
```
**Resultado:** cria `ano_cotacao`, `mes_cotacao` e `dias_para_vencer` (diferença em
dias). Esses derivados habilitam os agrupamentos temporais e a segmentação
curto/longo prazo usados nas etapas 4 e 5.

> Observação: aqui é uma consulta de _demonstração_ (com `LIMIT 20`). Para uso
> recorrente, vale materializar os derivados (p. ex. `ALTER TABLE ... ADD COLUMN` +
> `UPDATE`, ou uma `VIEW`), em vez de recalcular a cada consulta.

---

## 4. Análise descritiva dos dados

### Consulta 11 — Estatísticas descritivas das taxas de venda
```sql
SELECT
    tipo_titulo,
    ROUND(MIN(taxa_venda_manha), 2) AS menor_taxa_venda,
    ROUND(MAX(taxa_venda_manha), 2) AS maior_taxa_venda,
    ROUND(AVG(taxa_venda_manha), 2) AS taxa_venda_media,
    COUNT(*) AS total_cotacoes
FROM tesouro_direto
WHERE taxa_venda_manha IS NOT NULL
GROUP BY tipo_titulo
ORDER BY taxa_venda_media DESC;
```
**Resultado:**

| tipo_titulo | mín | máx | média | n |
|---|---:|---:|---:|---:|
| Tesouro Prefixado c/ Juros Sem. | 4,36 | 16,31 | 11,88 | 6.670 |
| Tesouro Prefixado | 2,89 | 16,33 | 11,76 | 6.205 |
| Tesouro Educa+ | 4,96 | 8,47 | 6,72 | 10.369 |
| Tesouro Renda+ | 5,33 | 7,80 | 6,53 | 5.832 |
| Tesouro IPCA+ | 1,73 | 10,49 | 6,24 | 5.840 |
| Tesouro IPCA+ c/ Juros Sem. | 1,64 | 10,49 | 6,13 | 10.816 |
| Tesouro IGPM+ c/ Juros Sem. | **-3,28** | 8,23 | 5,73 | 1.306 |
| Tesouro Selic | **-0,04** | 0,36 | 0,10 | 5.668 |

Prefixados pagam as maiores taxas **nominais** médias (~11,8%); IPCA+ entrega taxa
**real** (acima da inflação). Os valores **negativos** em IGPM+ e Selic merecem
atenção (ver seção 6).

### Consulta 12 — Estatísticas descritivas dos Preços Unitários (PU)
```sql
SELECT
    tipo_titulo,
    ROUND(MIN(pu_venda_manha), 2) AS menor_preco_unitario,
    ROUND(MAX(pu_venda_manha), 2) AS maior_preco_unitario,
    ROUND(AVG(pu_venda_manha), 2) AS preco_unitario_medio
FROM tesouro_direto
WHERE pu_venda_manha IS NOT NULL
GROUP BY tipo_titulo
ORDER BY preco_unitario_medio DESC;
```
**Resultado (destaques):** Selic é o PU mais alto (média ~R$ 13.765); o Prefixado
(zero-cupom) é o mais barato (média ~R$ 744). **Tesouro IGPM+ aparece com PU mínimo
de R$ 0,00** — valor inválido (ver seção 6).

### Consulta 13 — SELIC média por ano (diária)
```sql
SELECT
    EXTRACT(YEAR FROM data) AS ano_calendario,
    ROUND(MIN(valor), 2) AS selic_minima_ano,
    ROUND(MAX(valor), 2) AS selic_maxima_ano,
    ROUND(AVG(valor), 2) AS selic_media_ano
FROM selic_diaria
GROUP BY ano_calendario
ORDER BY ano_calendario;
```
**Resultado:** média diária `0,02` (2021) → `0,05` (2022–2023) → `0,04` (2024) →
`0,05` (2025). Mapeia o ciclo: estímulo monetário em 2021 e aperto a partir de 2022.

### Consulta 14 — Volatilidade de preço por vencimento (prefixados)
```sql
SELECT
    tipo_titulo,
    data_vencimento,
    ROUND(MIN(pu_venda_manha), 2) AS preco_minimo,
    ROUND(MAX(pu_venda_manha), 2) AS preco_maximo,
    ROUND(MAX(pu_venda_manha) - MIN(pu_venda_manha), 2) AS amplitude_variacao_preco
FROM tesouro_direto
WHERE tipo_titulo LIKE 'Tesouro Prefixado%'
GROUP BY tipo_titulo, data_vencimento
ORDER BY tipo_titulo, data_vencimento;
```
**Resultado (padrão claro):** quanto **mais distante o vencimento, maior a amplitude
de preço**. Ex.: Prefixado venc. 2022 → amplitude R$ 35,59; venc. 2026 → R$ 393,19.
Comprova que títulos longos são mais sensíveis (duration maior).

### Consulta 15 — Assimetria: 10 dias de maior taxa IPCA+
```sql
SELECT
    t.data_base,
    t.tipo_titulo,
    t.taxa_venda_manha AS taxa_titulo,
    s.valor AS taxa_selic_dia
FROM tesouro_direto t
JOIN selic_diaria s ON t.data_base = s.data
WHERE t.tipo_titulo LIKE 'Tesouro IPCA+%'
ORDER BY t.taxa_venda_manha DESC
LIMIT 10;
```
**Resultado:** os 10 maiores prêmios do IPCA+ (taxas de 10,43% a 10,49%) ocorrem
todos em **dezembro/2025**, exatamente quando a SELIC diária estava no pico
(`0,055131`). Os picos de juro real coincidem com o auge do aperto monetário.

---

## 5. Análise orientada pelos objetivos

### Bloco 1 — Objetivo 1 (poder de compra e Tesouro IPCA+)

#### Consulta 16 — Evolução mensal: SELIC vs taxa do IPCA+
```sql
SELECT
    EXTRACT(YEAR  FROM s.data) AS ano,
    EXTRACT(MONTH FROM s.data) AS mes,
    ROUND(AVG(s.valor), 4)            AS taxa_selic_diaria_media,
    ROUND(AVG(t.taxa_venda_manha), 2) AS taxa_venda_media_ipca
FROM selic_diaria s
JOIN tesouro_direto t ON s.data = t.data_base
WHERE t.tipo_titulo LIKE 'Tesouro IPCA+%'
  AND t.taxa_venda_manha IS NOT NULL
GROUP BY ano, mes
ORDER BY ano, mes;
```
**Resultado:** série mensal (60 meses). Conforme a SELIC sobe (de ~0,0075 em jan/2021
para ~0,0551 em 2025), a taxa real do IPCA+ acompanha (de ~3,3% para ~7,8%).
Tabela ideal para um gráfico de linhas comparando as duas curvas.

#### Consulta 17 — IPCA+ por cenário de SELIC
```sql
SELECT
    CASE
        WHEN s.valor >= 0.05 THEN '1. Dias de Selic Alta (>= 0.05)'
        WHEN s.valor <= 0.02 THEN '3. Dias de Selic Baixa (<= 0.02)'
        ELSE '2. Dias de Selic Intermediária'
    END AS cenario_selic,
    ROUND(AVG(s.valor), 4)            AS selic_diaria_media,
    ROUND(AVG(t.taxa_venda_manha), 2) AS taxa_media_ipca_mais,
    ROUND(MAX(t.taxa_venda_manha), 2) AS taxa_maxima_ipca_mais
FROM selic_diaria s
JOIN tesouro_direto t ON s.data = t.data_base
WHERE t.tipo_titulo LIKE 'Tesouro IPCA+%'
  AND t.taxa_venda_manha IS NOT NULL
GROUP BY cenario_selic
ORDER BY cenario_selic;
```
**Resultado:**

| cenário | SELIC diária média | taxa média IPCA+ | taxa máx IPCA+ |
|---|---:|---:|---:|
| Selic Alta (≥ 0,05) | 0,0526 | **6,88%** | 10,49% |
| Selic Intermediária | 0,0420 | 6,19% | 10,39% |
| Selic Baixa (≤ 0,02) | 0,0129 | **3,98%** | 5,12% |

**Resposta ao Objetivo 1:** confirmado. Em dias de SELIC baixa, o prêmio médio do
IPCA+ foi de **3,98%**; em dias de SELIC alta, saltou para **6,88%** (≈ o dobro), com
picos de **10,49%**. O Tesouro paga prêmios reais bem maiores quando os juros estão
altos, para atrair o investidor.

### Bloco 2 — Objetivo 2 (efeito gangorra e Tesouro Prefixado)

#### Consulta 18 — Evolução mensal: SELIC vs PU do Prefixado
```sql
SELECT
    EXTRACT(YEAR  FROM s.data) AS ano,
    EXTRACT(MONTH FROM s.data) AS mes,
    ROUND(AVG(s.valor), 4)          AS taxa_selic_diaria_media,
    ROUND(AVG(t.pu_venda_manha), 2) AS pu_medio_prefixado
FROM selic_diaria s
JOIN tesouro_direto t ON s.data = t.data_base
WHERE t.tipo_titulo LIKE 'Tesouro Prefixado%'
  AND t.pu_venda_manha IS NOT NULL
GROUP BY ano, mes
ORDER BY ano, mes;
```
**Resultado:** série mensal de 60 meses mostrando o movimento inverso — quando a
SELIC sobe, o PU médio do prefixado cai. (Ver ressalva metodológica abaixo.)

#### Consulta 19 — Correlação anual de extremos (pico da SELIC vs PU mínimo)
```sql
SELECT
    EXTRACT(YEAR FROM s.data) AS ano,
    ROUND(MAX(s.valor), 4)          AS selic_diaria_maxima,
    ROUND(MIN(t.pu_venda_manha), 2) AS pu_minimo_prefixado,
    ROUND(AVG(t.pu_venda_manha), 2) AS pu_medio_prefixado
FROM selic_diaria s
JOIN tesouro_direto t ON s.data = t.data_base
WHERE t.tipo_titulo LIKE 'Tesouro Prefixado%'
  AND t.pu_venda_manha IS NOT NULL
GROUP BY ano
ORDER BY ano;
```
**Resultado:**

| ano | SELIC diária máx | PU mín prefixado | PU médio prefixado |
|---|---:|---:|---:|
| 2021 | 0,0347 | 605,69 | 933,70 |
| 2022 | 0,0508 | 437,81 | 848,09 |
| 2023 | 0,0508 | 470,20 | 873,87 |
| 2024 | 0,0455 | 417,16 | 864,80 |
| 2025 | 0,0551 | 381,26 | 773,10 |

**Resposta ao Objetivo 2:** confirmado o "efeito gangorra". Em 2021 (SELIC mais
baixa) o PU médio era o mais alto (R$ 933,70); em 2025 (pico da SELIC) caiu para
R$ 773,10, com mínima histórica de **R$ 381,26**. Juros sobem → preço dos títulos
antigos cai.

#### Consulta 20 — Amplitude de preço dos prefixados por vencimento
```sql
SELECT
    t.tipo_titulo,
    t.data_vencimento,
    ROUND(MIN(t.pu_venda_manha), 2) AS menor_preco_unitario,
    ROUND(MAX(t.pu_venda_manha), 2) AS maior_preco_unitario,
    ROUND(MAX(t.pu_venda_manha) - MIN(t.pu_venda_manha), 2) AS amplitude_preco_no_ciclo
FROM selic_diaria s
JOIN tesouro_direto t ON s.data = t.data_base
WHERE t.tipo_titulo LIKE 'Tesouro Prefixado%'
GROUP BY t.tipo_titulo, t.data_vencimento
ORDER BY t.data_vencimento ASC;
```
**Resultado:** reforça que o efeito gangorra é mais severo nos vencimentos longos
(amplitudes que chegam a R$ 418,51 no Prefixado c/ Juros venc. 2031).

> ⚠️ Esta consulta é praticamente idêntica à **Consulta 14**, e o `JOIN selic_diaria`
> é **redundante** (nenhuma coluna de `s` é usada). Ver seção 6.

---

## 6. Síntese dos achados

- **Objetivo 1 (IPCA+):** o prêmio real médio do IPCA+ quase **dobra** entre cenários
  de juros baixos (3,98%) e altos (6,88%), confirmando que o Tesouro eleva o prêmio
  para proteger/atrair o investidor em ciclos de aperto.
- **Objetivo 2 (Prefixado):** confirmado o "efeito gangorra" — o PU cai à medida que a
  SELIC sobe, e o efeito é proporcional ao prazo de vencimento (títulos longos sofrem
  variações muito maiores).
- **Qualidade dos dados:** chave primária íntegra (sem duplicatas), cronologia
  consistente, colunas de venda sem nulos.

### Pontos de atenção / limitações encontradas

1. **Mistura de subtipos de prefixado.** As consultas 18 e 19 usam
   `LIKE 'Tesouro Prefixado%'`, que captura **dois instrumentos diferentes** —
   "Tesouro Prefixado" (zero-cupom, PU ~R$ 380–1.000) e "Tesouro Prefixado com Juros
   Semestrais" (PU ~R$ 745–1.213). O `AVG(pu_venda_manha)` mistura os dois, então o
   "PU médio do prefixado" deve ser lido com cautela. _Sugestão:_ usar
   `tipo_titulo = 'Tesouro Prefixado'` quando o foco for o zero-cupom.
2. **Valores anômalos não tratados no Tesouro.** A consulta 12 mostra
   **PU mínimo = R$ 0,00** no IGPM+ (valor inválido/placeholder); a consulta 11 mostra
   **taxas negativas** (IGPM+ -3,28%; Selic -0,04%). A verificação de domínio
   (consulta 8) cobriu apenas a SELIC — o domínio de preços/taxas do Tesouro ficou de
   fora.
3. **Consulta 20 redundante.** Repete a consulta 14 e faz um `JOIN` inútil com
   `selic_diaria`. Pode ser removida ou reescrita para realmente cruzar com a SELIC.
4. **Verificação de nulos parcial.** A consulta 6 só testou colunas de _venda_; os
   nulos previstos na Parte 1 estão nas colunas de _compra_, não verificadas.

---

## 7. Análise crítica das fontes de dados

Enquanto a seção 6 trata das limitações das **consultas** desta análise, esta seção
registra as restrições estruturais das **próprias fontes** — características que
condicionam o que é (e o que não é) possível concluir com os dados.

### Taxa SELIC Diária (BACEN — Série SGS nº 11)

- **Cobertura apenas de dias úteis bancários:** a ausência de entradas em fins de semana
  e feriados exige cuidado em análises de intervalos de tempo contínuos e no cálculo de
  retornos acumulados.
- **Granularidade:** a série representa a taxa efetiva média do dia, publicada pelo BACEN
  com atraso de um dia útil. Não captura valores ao longo do dia para análises intraday
  e não reflete expectativas futuras.
- **Limitação estrutural (diária × anualizada):** a série utilizada (SGS nº 11) representa
  a taxa SELIC efetiva **diária** (% ao dia), e não a taxa SELIC "oficial" divulgada pelo
  Copom, que é anualizada considerando 252 dias úteis. Comparações ou conversões entre as
  duas convenções (diária ↔ anualizada) devem ser feitas com cuidado para evitar
  interpretações equivocadas dos valores.

### Preços e Taxas do Tesouro Direto (Tesouro Nacional)

- **Nulos nas taxas de compra:** alguns títulos têm `taxa_compra_manha` nula em dias em
  que o Tesouro suspendeu a compra (operações de recompra suspensas). Isso não é erro, é
  limitação do produto.
- **Cotações apenas da manhã:** o arquivo não disponibiliza cotações ao longo do pregão.
  Análises de variação de valores dentro de um mesmo dia são impossíveis com esta fonte.
- **Descontinuidade de títulos:** títulos emitidos antes de 2021 e retirados de circulação
  antes do período analisado não aparecem, criando **viés de sobrevivência** para análises
  de longo prazo.
- **Heterogeneidade dos tipos de título:** a coluna `tipo_titulo` mistura informações de
  categoria (IPCA+, Prefixado, Selic), produto (Renda+, Educa+) e vencimento em alguns
  casos, dificultando agrupamentos sem normalização prévia.
- **Ausência de volume negociado:** os dados de preços e taxas não incluem volume
  financeiro, impedindo análises de liquidez efetiva.

---

## 8. Mapa de arquivos

| Pasta / arquivo | Conteúdo |
|---|---|
| `docs/Relatorio_Parte1_IBD.pdf` | Relatório da Parte 1 (modelagem, dicionário de dados e análise das fontes) |
| `docs/consultas-resultados/consulta_1..20` | Resultados exportados das consultas em CSV (faltam 7, 8 e 9 — retornaram vazio) |
| `docs/Apresentacao_TP-IBD.pptx` | Slides da apresentação do trabalho |

### Correspondência consulta → arquivo de resultado

| # | Consulta | CSV |
|---|---|---|
| 1–5 | Caracterização | `consulta_1` … `consulta_5` |
| 6 | Nulos | `consulta_6` |
| 7 | Duplicatas | _(vazio — sem CSV)_ |
| 8 | Domínio SELIC | _(vazio — sem CSV)_ |
| 9 | Cronologia | _(vazio — sem CSV)_ |
| 10 | Atributos derivados | `consulta_10` |
| 11–15 | Descritiva | `consulta_11` … `consulta_15` |
| 16–20 | Objetivos | `consulta_16` … `consulta_20` |

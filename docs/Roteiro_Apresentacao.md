# Roteiro de Apresentação — TP IBD (SELIC e Tesouro Direto)

> Base: `Apresentacao_TP-IBD.pptx` (13 slides). Os números deste roteiro vêm de
> `docs/Relatorio_Parte2.md` (fonte da verdade). Tempo-alvo: ~10–12 min.

## Divisão de falas

| Slides | Bloco | Responsável |
|---|---|---|
| 1–2 | Abertura: capa e roteiro | **Enzo** |
| 3 | Fontes de dados: SELIC, Tesouro Direto e PU | **Fernando** |
| 4–5 | Modelagem (Parte 1) + Metodologia (Parte 2) | **Davi** |
| 6–7 | Caracterização + introdução aos objetivos | **Fernando** |
| 8–9 | Preparação dos dados + Análise descritiva | **Enzo** |
| 10–11 | Objetivo 1 e Objetivo 2 | **Pedro Puliti** |
| 12–13 | Análise crítica + Conclusões e fechamento | **Otávio** |

Carga: Enzo 4 slides · Fernando 3 · Otávio 2 · Puliti 2 · Davi 2.

---

## Enzo — Abertura (slides 1 e 2)

### Slide 1 — Capa
> "Bom dia/boa tarde. Nosso trabalho integra duas fontes públicas: a **taxa SELIC diária**,
> do Banco Central, e os **preços e taxas do Tesouro Direto**, do Tesouro Nacional. A ideia é
> investigar como as decisões de juros do Banco Central afetam os títulos públicos."

### Slide 2 — Roteiro
> "Vamos passar por: fontes e motivação, a modelagem da Parte 1, a metodologia da Parte 2 com
> 20 consultas SQL, os principais achados, a análise crítica e as conclusões. Começando pelas
> fontes, passo para o Fernando."

---

## Fernando — Fontes de dados (slide 3)

### Slide 3 — Fontes de dados (introduzir SELIC, Tesouro e PU)
> "A **SELIC** é a taxa básica de juros da economia. Usamos a série diária do BACEN (SGS nº 11):
> **1.256 registros**, de 2021 a 2025, em dias úteis. Importante: é a taxa **efetiva diária**
> (% ao dia), não a anualizada do Copom.
>
> O **Tesouro Direto** é o programa de venda de títulos públicos a pessoas físicas. São
> **52.706 cotações**, de 8 famílias de títulos. Cada título tem uma **taxa** (% ao ano) e um
> **Preço Unitário, o PU** — que é, em reais, quanto custa o título naquele dia. Guardem o conceito
> de PU, porque ele é central no Objetivo 2. Sobre a modelagem disso, passo para o Otávio."

---

## Davi — Modelagem e metodologia (slides 4 e 5)

### Slide 4 — Parte 1: Modelagem
> "Modelamos duas tabelas: `selic_diaria` e `tesouro_direto`, no PostgreSQL. A junção é pela data:
> `tesouro_direto.data_base` referencia `selic_diaria.data` (chave estrangeira). A chave primária do
> Tesouro é composta — tipo de título, vencimento e data de cotação. São 8 tipos de título no total."

### Slide 5 — Parte 2: Metodologia
> "A análise foi feita direto em SQL no PostgreSQL: **20 consultas em 5 etapas** —
> caracterização, definição dos objetivos, preparação, análise descritiva e análise por objetivos.
> Volto ao Fernando para a caracterização e as perguntas de pesquisa."

---

## Fernando — Caracterização e objetivos (slides 6 e 7)

### Slide 6 — Etapa 1: Caracterização
> "No total são **53.962 registros** integrados, cobrindo 2021 a 2025. A SELIC diária variou de
> **0,0075% a 0,0551% ao dia** — algo como ~2% a ~15% ao ano: um período de forte aperto monetário.
> E a maior taxa de venda registrada foi **16,33%**, num Tesouro Prefixado."

### Slide 7 — Objetivos de pesquisa (introduzir as perguntas)
> "Com os dados caracterizados, definimos duas perguntas de pesquisa:
> **Objetivo 1** — em ciclos de alta da SELIC, o Tesouro paga prêmios maiores nos títulos
> **IPCA+** para atrair o investidor?
> **Objetivo 2** — como a alta e a queda da SELIC afetam o **Preço Unitário** dos títulos
> **Prefixados**?
> Antes de responder, o Enzo mostra como preparamos e descrevemos os dados."

---

## Enzo — Preparação e análise descritiva (slides 8 e 9)

### Slide 8 — Etapa 3: Preparação dos dados
> "Antes de analisar, validamos a qualidade: **zero duplicatas** (a chave primária composta garante
> unicidade), **zero nulos** nas colunas de venda, **cronologia consistente** (nenhum vencimento
> antes da cotação) e criamos atributos derivados (ano, mês, dias para vencer).
> Um ponto de atenção: achamos valores fora do domínio no Tesouro — **PU de R$ 0,00** no IGPM+ e
> **taxas negativas** (IGPM+ −3,28%; Selic −0,04%) — que retomamos na análise crítica."

### Slide 9 — Etapa 4: Análise descritiva
> "Na descritiva, os **Prefixados** pagam as maiores taxas nominais médias (~11,8%); o **IPCA+**
> entrega taxa real, acima da inflação. E fica visível o ciclo: 2021 com juros baixos e, a partir
> de 2022, aperto monetário sustentado. As taxas em vermelho são as negativas, que extrapolam o
> domínio esperado. Com isso, o Puliti responde às duas perguntas."

---

## Pedro Puliti — Objetivos 1 e 2 (slides 10 e 11)

### Slide 10 — Etapa 5, Objetivo 1: prêmio do IPCA+ vs. SELIC
> "Retomando o Objetivo 1: o Tesouro paga mais no IPCA+ quando os juros sobem?
> **Confirmado.** Separando os dias por cenário de SELIC, o prêmio real médio do IPCA+ **quase
> dobra**: **3,98%** em dias de juros baixos contra **6,88%** em dias de juros altos, com picos de
> **10,49%** no auge do aperto, em dezembro de 2025. Ou seja, o governo eleva o prêmio para atrair
> o investidor em ciclos de aperto."

### Slide 11 — Etapa 5, Objetivo 2: efeito gangorra no Prefixado
> "No Objetivo 2, o **efeito gangorra**: juros sobem, preço do título antigo cai.
> **Confirmado.** O PU médio do Prefixado caiu de **R$ 933,70 em 2021** (SELIC mais baixa) para
> **R$ 773,10 em 2025** (pico da SELIC), com mínima histórica de **R$ 381,26**. E o efeito é mais
> forte nos títulos de vencimento **mais longo**, que são mais sensíveis. Passo para o Davi, que
> fecha com a análise crítica e as conclusões."

---

## Otávio — Análise crítica e conclusões (slides 12 e 13)

### Slide 12 — Análise crítica (penúltimo)
> "Antes de concluir, é importante sermos honestos sobre os limites das fontes.
>
> Na **SELIC**: ela cobre só dias úteis, é publicada com um dia de atraso e não tem granularidade
> intradiária. E é a série **diária** (SGS 11), diferente da SELIC oficial anualizada do Copom —
> por isso todo cuidado ao converter entre as duas convenções.
>
> No **Tesouro Direto**: só temos a cotação da manhã, então não dá para analisar variação ao longo
> do dia; há **viés de sobrevivência** (títulos descontinuados antes de 2021 não aparecem); a coluna
> `tipo_titulo` mistura categoria, produto e vencimento, exigindo normalização; e **não há dados de
> volume** negociado, então não conseguimos medir liquidez.
>
> Foi também aqui que tratamos as anomalias do slide de preparação — os PU zerados e as taxas
> negativas — como limitações de domínio, não como conclusões."

### Slide 13 — Conclusões (fechamento)
> "Concluindo: **política monetária e precificação de títulos andam juntas.**
> No **Objetivo 1**, o prêmio real do IPCA+ quase dobra (3,98% → 6,88%) quando a SELIC sobe — o
> governo paga mais para atrair investidores no aperto.
> No **Objetivo 2**, confirmamos o efeito gangorra: o PU do Prefixado cai quando a SELIC sobe, e mais
> ainda nos vencimentos longos.
> Sobre **qualidade dos dados**: chave primária íntegra, cronologia consistente e colunas de venda
> sem nulos, com os pontos de atenção documentados.
> Tudo — dados, modelo ER, dicionário, análise crítica e relatórios — está publicado no nosso
> repositório no GitHub. Obrigado, ficamos à disposição para perguntas."

---

## Transições (quem entrega para quem)

`Enzo (1–2) → Fernando (3) → Davi (4–5) → Fernando (6–7) → Enzo (8–9) → Puliti (10–11) → Otávio (12–13)`

Cada slide final de bloco já tem a deixa de passagem embutida na fala. Números-chave para decorar:
**53.962 registros**, **3,98% → 6,88%** (Obj. 1), **R$ 933,70 → R$ 773,10** (Obj. 2),
**16,33%** (maior taxa).

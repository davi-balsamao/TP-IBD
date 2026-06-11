Análise Crítica das Fontes de Dados

Taxa SELIC Diária (BACEN — Série SGS nº 11)
•	Cobertura apenas de dias úteis bancários: ausência de entradas em fins de semana e feriados exige cuidado em análises de intervalos de tempo contínuos e no cálculo de retornos acumulados.
•	Granularidade: a série representa a taxa efetiva média do dia, publicada pelo BACEN com atraso de um dia útil. Não captura valores ao longo do dia para análises intraday e não reflete expectativas futuras.
•	Limitação estrutural: a série utilizada (SGS nº 11) representa a taxa SELIC efetiva diária (% ao dia), e não a taxa SELIC "oficial" divulgada pelo Copom, que é anualizada considerando 252 dias úteis. Comparações ou conversões entre as duas convenções (diária ↔ anualizada) devem ser feitas com cuidado para evitar interpretações equivocadas dos valores.

Preços e Taxas do Tesouro Direto (Tesouro Nacional)
•	Nulos nas taxas de compra: alguns títulos têm taxa_compra_manha nula em dias em que o Tesouro suspendeu a compra (operações de recompra suspensas). Isso não é erro, é limitação do produto.
•	Cotações apenas da manhã: o arquivo não disponibiliza cotações ao longo do pregão. Análises de valores em um mesmo dia são impossíveis com esta fonte.
•	Descontinuidade de títulos: títulos emitidos antes de 2021 e retirados de circulação antes do período analisado não aparecem, criando viés de sobrevivência para análises de longo prazo.
•	Heterogeneidade dos tipos de título: a coluna tipo_titulo mistura informações de categoria (IPCA+, Prefixado, Selic), produto (Renda+, Educa+) e vencimento em alguns casos, dificultando agrupamentos sem normalização prévia.
•	Ausência de volume negociado: os dados de preços e taxas não incluem volume financeiro, impedindo análises de liquidez efetiva.
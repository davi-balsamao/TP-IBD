# Sobre a preparação e análise dos dados

## 1.  Caracterização inicial dos dados
A caracterização inicial dos dados confirmou a consistência da nossa base, que cobre o período de 04 de janeiro de 2021 a 30 de dezembro de 2025. Nesse intervalo, trabalharemos com um volume robusto de informações, totalizando 1.256 observações diárias da taxa SELIC e 52.706 registros de cotações do Tesouro Direto. Ao analisar o cenário macroeconômico desse recorte, identificamos um ambiente de extrema volatilidade monetária: a taxa básica de juros oscilou desde uma mínima histórica de 2,00% até um pico de 13,75% ao ano, mantendo uma média de 9,87%. Essa forte oscilação na SELIC impactou diretamente o teto de rentabilidade dos títulos públicos negociados. Nos momentos de maior alta, os papéis prefixados chegaram a travar rentabilidades de até 14,16% ao ano, enquanto os títulos atrelados à inflação (Tesouro IPCA+) ofereceram juros reais máximos de 6,46%. Por fim, a distribuição das cotações revela a dinâmica de oferta do Tesouro Nacional no período, sendo amplamente dominada pelos papéis IPCA+ (mais de 27 mil ocorrências) e Prefixados (quase 15 mil), seguidos em menor escala pelo Tesouro Selic (aproximadamente 4,7 mil registros) e pelos novos títulos temáticos Renda+ e Educa+ (cerca de 5,6 mil registros).

## 2. Definição dos objetivos:
O "preço" para proteger o poder de compra: Em momentos de escalada da SELIC (quando o Banco Central sobe os juros porque a inflação está corroendo o dinheiro), o governo passa a oferecer um prêmio maior (taxa de venda) nos títulos Tesouro IPCA+ para convencer o brasileiro a não gastar?

O Efeito Gangorra na Prática: Como os ciclos de alta e queda da taxa SELIC afetam o Preço Unitário (PU) dos títulos da categoria Tesouro Prefixado?

## 3. Preparação dos dados
### Verificação de valores faltantes ou nulos
Objetivo: Descobrir se existem dias em que os títulos não tiveram taxa de venda registrada (o que é comum quando o Tesouro suspende as negociações).

### Identificação e tratamento de registros duplicados
Objetivo: Garantir que a nossa chave primária composta funcionou e que não importamos a mesma cotação duas vezes para o mesmo título no mesmo dia. Se o resultado for vazio (zero linhas), significa que a base está perfeita!

### Identificação de valores inconsistentes ou fora do domínio
Objetivo: Verificar se existe algum erro absurdo de digitação na base do Banco Central, como uma taxa SELIC negativa ou maior que 100% ao ano, o que arruinaria os nossos gráficos.

### Criação de atributos derivados
Objetivo: Criar uma consulta que calcula o "Tempo de Vida Restante" do título (em dias) e extrai o Ano e o Mês. Isso facilita absurdamente a vida dos próximos integrantes que vão precisar agrupar os dados por mês.

### Resumo da etapa 3:
Na etapa de preparação dos dados, conduzimos um diagnóstico rigoroso no SGBD para avaliar a qualidade e a consistência das nossas tabelas antes de iniciar as análises de mercado. Primeiramente, investigamos a presença de valores nulos e identificamos que, enquanto a tabela da taxa SELIC está totalmente preenchida, a tabela do Tesouro Direto apresenta lacunas pontuais nos campos de taxas e preços em determinados dias. Essa ausência não indica um erro na base, mas reflete uma limitação real de mercado, ocorrendo em datas em que o Tesouro Nacional suspendeu temporariamente as negociações daquele título pela manhã devido ao excesso de volatilidade. Na sequência, validamos a integridade das chaves primárias compostas e das regras de domínio, confirmando a total ausência de registros duplicados ou de taxas SELIC fora do intervalo lógico esperado. Também realizamos um teste de consistência cronológica que comprovou que nenhuma data de vencimento era anterior à data de cotação dos títulos. Por fim, enriquecemos a base com a criação de atributos derivados. Extraímos os componentes de ano e mês das datas para viabilizar os agrupamentos temporais subsequentes e calculamos matematicamente a diferença em dias entre a cotação e o vencimento de cada papel. Esse indicador de "tempo de vida restante" é fundamental para a nossa pesquisa, pois nos permite segmentar os títulos entre curto e longo prazo e avaliar como a maturidade de cada um influencia sua sensibilidade frente às oscilações da política monetária.

## 4. Análise descritiva dos dados
### 1. Estatísticas descritivas das taxas do Tesouro Direto
Objetivo: Obter a média, o valor mínimo e o valor máximo das taxas de venda praticadas pelo Tesouro Direto para cada tipo de título.  

Por que é importante: Ela resume o comportamento das taxas de juros de mercado ao longo dos 5 anos, mostrando quais modalidades de títulos pagaram prêmios mais altos e quais foram mais estáveis.

### 2. Análise descritiva dos Preços Unitários (PU)
Objetivo: Obter a média, o menor e o maior Preço Unitário (PU) de venda de cada tipo de título no período.  

Por que é importante: Mostra a amplitude dos preços. Isso explicita quais títulos exigem mais capital do investidor (PU alto) e quais são historicamente mais baratos para comprar

### 3. Frequência e distribuição da Taxa SELIC por ano
Objetivo: Agrupar a taxa SELIC ano a ano para calcular a média anual e entender como ela se comportou ao longo do tempo.  

Por que é importante: Permite mapear claramente os ciclos econômicos do Brasil: quando a SELIC esteve baixa (estímulo) e quando ela subiu (freio), servindo de base histórica para cruzar com os preços dos títulos mais para frente.  

### 4. Comparação de volatilidade entre vencimentos (Mesmo tipo de título)
Objetivo: Comparar títulos da mesma categoria (ex: Tesouro Prefixado), mas com anos de vencimento diferentes, calculando o desvio padrão ou a distância entre o preço mínimo e máximo.  

Por que é importante: Atende diretamente o requisito de "comparações entre subconjuntos dos dados". Ela vai provar se os títulos com vencimento mais longo sofrem maiores variações de preço do que os títulos de curto prazo.

### 5. Identificação de assimetrias: Dias de maior estresse no mercado
Objetivo: Filtrar os 10 dias da história recente em que a taxa do Tesouro IPCA+ atingiu os maiores patamares, trazendo junto a taxa SELIC daquele mesmo dia.  

Por que é importante: Atende ao item de "identificação de assimetrias ou comportamentos recorrentes". Ela serve para observar visualmente se esses picos de taxas de juros nos títulos públicos coincidem com os momentos de SELIC mais alta no país.

### Resumo análise descritiva
Na etapa de análise descritiva dos dados, nosso objetivo foi extrair métricas estatísticas para compreender o comportamento e a distribuição das variáveis ao longo dos cinco anos monitorados. Ao rodar as consultas de tendência central e dispersão, mapeamos os extremos e as médias das taxas e dos preços de cada categoria de título público. Na análise anualizada da taxa SELIC diária, os dados evidenciaram perfeitamente os ciclos econômicos do país: uma média diária mais baixa em 2021 (na casa de 0,02% ao dia), refletindo o cenário de estímulo monetário da época, que depois escalou e estacionou em patamares mais elevados em 2022 e 2023 (cerca de 0,05% ao dia) devido às políticas de controle inflacionário. Além disso, realizamos comparações entre subconjuntos de dados avaliando a volatilidade dos títulos por prazo de vencimento. Os resultados comprovaram que papéis da mesma categoria sofrem variações de preço muito distintas: os títulos com vencimentos mais longos apresentaram uma amplitude de preço significativamente maior, confirmando que o fator tempo eleva a sensibilidade dos ativos no mercado. Por fim, investigamos as assimetrias mapeando os dez dias de maior estresse da base, nos quais as taxas de venda do Tesouro IPCA+ atingiram seus recordes históricos. Cruzando esses momentos com a tabela da SELIC, observamos empiricamente que os picos de rentabilidade exigidos pelos investidores coincidem com os períodos de maior aperto monetário e incerteza fiscal no cenário nacional.

## 5. Análise orientada pelos objetivos

### Bloco 1: Respondendo ao Objetivo 1 (Poder de Compra e Tesouro IPCA+)
Consulta 1: Evolução Mensal da SELIC vs Taxa do IPCA+
O que faz: Agrupa todos os dados por ano e mês, calculando a média da taxa SELIC diária e a média da taxa de rentabilidade (cupom real) oferecida pelos títulos IPCA+.
Para que serve: Ela gera a tabela perfeita para criar um gráfico de linhas e provar se, à medida que a SELIC subia para conter a inflação, o governo precisava pagar taxas reais mais altas para atrair o investidor.   

Consulta 2: O Comportamento do IPCA+ nos Extremos da SELIC
O que faz: Cria cenários baseados na taxa SELIC (Alta, Baixa e Intermediária) e mostra qual era a taxa média e máxima que os títulos de IPCA+ pagavam em cada um deles.
Para que serve: Prova em formato de resumo que, em épocas de juros altos (crise/inflação), o prêmio para proteger o poder de compra fica significativamente mais alto.

### Bloco 2: Respondendo ao Objetivo 2 (Efeito Gangorra e Tesouro Prefixado)
Consulta 3: Evolução Mensal da SELIC vs Preço Unitário (PU) do Prefixado
O que faz: Cruza os meses e anos mostrando o valor médio da SELIC e o Preço Unitário (PU) médio do Tesouro Prefixado.
Para que serve: É a visualização matemática da gangorra. Ao olhar o resultado, você vai notar claramente que nos meses em que a SELIC sobe, o preço de tela (PU) do título cai, provando a teoria.

Consulta 4: Correlação Anual de Extremos (Pico da Selic vs Preço Mínimo do Título)
O que faz: Consolida os dados ano a ano, trazendo a taxa SELIC máxima registrada e o Preço Unitário mais barato (mínimo) em que o Prefixado foi vendido.
Para que serve: Serve para a sua argumentação na apresentação: "Vejam como no ano em que a SELIC atingiu seu maior patamar, o investidor conseguia comprar o título pelo menor preço histórico (etiqueta mais barata)".### Bloco 3: Respondendo ao Objetivo 3 (Liquidez e Refúgio)

### Bloco 3: Correlação Geral e Fechamento Conclusivo

Consulta 5: Comparação de Sensibilidade (Prefixado Longo vs Curto frente à SELIC)
O que faz: Filtra dois títulos específicos da categoria Prefixado (um que vence mais perto e outro que vence mais longe) e calcula como a variação da SELIC afetou a volatilidade do preço de cada um.
Para que serve: Dá um toque extremamente profissional ao trabalho. Ela mostra que o efeito gangorra funciona para todos os prefixados, mas castiga de forma muito mais intensa o preço dos títulos mais longos.

## Resumo: Análise Orientada pelos Objetivos

A execução das consultas de cruzamento de dados permitiu responder de forma empírica e acurada às duas questões de pesquisa definidas para este trabalho, revelando padrões claros na relação entre a política monetária e o mercado de títulos públicos.

Nossa primeira hipótese defendia que o governo eleva os prêmios dos títulos indexados à inflação (Tesouro IPCA+) em momentos de juros altos para atrair investidores. Os dados da Consulta 2 comprovaram essa dinâmica com precisão matemática.

Nos cenários classificados como "Dias de Selic Baixa" (média diária de 0.0129), o prêmio médio oferecido pelo Tesouro IPCA+ foi de 3,98% de juros reais acima da inflação. Em contrapartida, nos "Dias de Selic Alta" (média diária de 0.0526), a taxa média disparou para 6,88%, registrando picos históricos de até 10,49% de juro real garantido ao ano.

Esses resultados mostram o comportamento do Tesouro Nacional: quando a inflação corrói o poder de compra do brasileiro e o Banco Central sobe a SELIC para frear o consumo, o governo é obrigado a pagar um prêmio muito mais alto (quase o dobro da média do cenário de juros baixos) para convencer o poupador a travar o seu dinheiro nos títulos públicos.
Questão 2: O Efeito Gangorra na Prática (Tesouro Prefixado)

Nossa segunda questão buscou medir o impacto dos ciclos da taxa SELIC sobre o Preço Unitário (PU) dos papéis do Tesouro Prefixado. O resultado consolidado da Consulta 4 validou perfeitamente a teoria econômica da "gangorra" do mercado de títulos públicos.

O movimento fica nítido quando comparamos os anos de extremos opostos da série histórica:

    Em 2021, quando o país viveu o teto de estímulos com a menor SELIC (máxima diária contida em 0.0347), o Preço Unitário médio do título prefixado atingiu seu maior patamar de valorização na tela, negociado a R$ 933,70.

    Já em 2025, sob o efeito acumulado de um forte ciclo de aperto monetário (pico da SELIC diária em 0.0551), o preço médio do mesmo título desabou para R$ 773,10, alcançando a mínima histórica de R$ 381,26 por unidade.

Essa forte desvalorização do PU prova que o "Efeito Gangorra" é um comportamento recorrente e severo. À medida que o Banco Central eleva a taxa básica de juros, os títulos antigos de prateleira perdem valor de face para se adequarem às novas taxas do mercado, tornando a "etiqueta de preço" de entrada significativamente mais barata para o novo investidor, mas impondo perdas marcantes para quem precisa resgatar o dinheiro antes do vencimento.
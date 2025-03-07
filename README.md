# CasaPrevendo

---
title: "Prevendo valores de Casas"
author: "Italo Iveldo"
date: "25/06/2020"
output:
  pdf_document: default
  html_document: default
  word_document: default
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

## Infomormações Gerais
Os dados coletados são referentes ao dataset disponibilizado no kagle por Rubens Junior (https://www.kaggle.com/rubenssjr/brasilian-houses-to-rent), possuem informações retiradas através de Webcrawler em diversos sites com ofertas de alugueis de imoveis. Os dados contém 10962 observações e 13 vaariaveis. Possuem as seguintes colunas:

city= Cidade;
area= Tamanho da casa em m2;
rooms= Número de quartos;
Bathroom= Número de banheiros;
Parking spaces= Número de vagas em estacionamento;
Floor= Número de andar;
Animal= Se aceita animais ou não;
Furniture= Se o local é mobiliado ou não;
Hoa= Valor do condominio;
rent amount (R$)= Valor do aluguel;
property tax (R$)= IPTU da casa;
fire insurance (R$)= Seguro de incendio da casa;
Total= Total pago ao mês pela casa;

### Objetivo:
Esse trabalho terá como objetivo prever valores futuros de aluguel utilizando regressão linear 

```{r Pacotes, message=FALSE, warning=FALSE}
#install.packages("readr")
#install.packages("plotly")
library(readr)
library(visdat)
library(dplyr)
library(ggplot2)
library(plotly)
library(dplyr)
library(corrplot)
library(randomForest)
library(caret)
```
```{r Importando, warning=TRUE}
df<- read_csv("houses_to_rent_v2.csv")
head(df)
```

### Mudança de nomes de variaveis
Com o objetivo de deixar mais simples a manipulação dos dados e o entendimento, decidimos mudar o nome das variaveis.

```{r Mudança de nomes}
names(df)= c("cidade","area","quartos","banheiros","n_estacionamento", "andar","animal","Furniture","condominio","valor_aluguel","IPTU","taxa_incendio","total_pago")
head(df)

```



### Tratando Dados

Inicialmente iremos avaliar se o dataset possuem algum valor faltante.Abaixo é possível perceber que não possuimos nenhum dado faltante, não precisando fazer qualquer tipo de tratamento.


```{r Tratar dados faltantes, warning=FALSE}
colSums(is.na(df))

```
Para começarmos a fazer qualquer tipo de analise, precisamos antes de tudo validar se as variaveis do nosso dataset estão formatadas com tipo de dados certo. Para ficar visualmente mais atrativo, decidimos utilizar o pacote visdat.

```{r Tipo de dado}
vis_dat(df)
```

Precisamos mudar os tipos das variaveis cidade, animal e furniture para o tipo fator. Embora realmente esses dados sejam strigs, elas representam uma variavel de classificação.

```{r mudança de variavel}
df$cidade= as.factor(df$cidade)
df$animal= as.factor(df$animal)
df$Furniture= as.factor(df$Furniture)
df$banheiros=as.integer(df$banheiros)
df
vis_dat(df)
```

### Analise Exploratoria

Com o gráfico abaixo é possível notar que a maioria das casas no dataset são da cidade de São Paulo 
```{r plot cidade}
g=ggplot(data=df, aes(x=cidade)) + geom_bar(fill="lightblue",)+ coord_flip()+
  theme_classic() + labs(title = "Número de casas para alugar por cidades") + ylab("numero de casas")
ggplotly(g)

```

Analisando agora os valores pagos em alugueis que é a variável que desejamos prever.Abaixo segue algumas analises estatisticas referentes a variavel valor do aluguel agrupadas por cidade
```{r Analise da variavel valor aluguel}
aggregate(df[,"valor_aluguel"], list(local=df$cidade), mean)
aggregate(df[,"valor_aluguel"], list(local=df$cidade), sd)
aggregate(df[,"valor_aluguel"], list(local=df$cidade), max)
aggregate(df[,"valor_aluguel"], list(local=df$cidade), min)
aggregate(df[,"valor_aluguel"], list(local=df$cidade), median)

```

Iremos avaliar agora como andam os valores outilines, é possivel já notar pelo desvio padrão que possuimos um conjunto de dados bastante esparço em relação a variavel valor_aluguel.Agora iremos visualizar através de um boxplot

```{r}
lista=c("banheiros","quartos","n_estacionamento")
legenda=c("Valor do aluguel pela quantidade de banheiros",
          "Valor do alguel pela quantidade de quartos",
          "Valor do aluguel pela quantidade de n_estacionamento")
plot.boxes<-function(X,legenda){
  ggplot(df,aes_string(x=X, y="valor_aluguel",group=X))+
      geom_boxplot() + labs(title = legenda)+
    theme_classic()
        
  
}

Map(plot.boxes,lista,legenda)
```


É possivel notar que as 3 variaveis possuem uma correlação com o valor do aluguel e possuem varios outlines.Decidimos não remove-los e deixa-los no modelo preditivo, pois sua remoção poderia criar um modelo que não representaria a realidade.

Finalizando a nossa analise exploratoria, abaixo segue uma gráfico que mostra a correlação entre as variaveis
```{r}
colnumericas=sapply(df, is.numeric)
correlacao= cor(df[,colnumericas])
corrplot(correlacao, method = "color",addCoef.col =T) 
```

É possivel notar uma correlação positiva média a forte entre as variaveis taxa de incendio, número de quartos, número de banheiros e número de vagas de estacionamento com a váriavel valor do aluguel. (Outra correlação muito forte é com o total pago, mas essa variavel não ajudará de nada nosso modelo, pois seu valor depende do preço do aluguel)

## Feature Select

Agora utilizaremos um modelo randomForest para criar um modelo com o objetivo de avaliar quais as variaveis mais importantes para prever o valor do aluguel
```{r}
modelo= randomForest(valor_aluguel~.-total_pago,df,ntree=100,nodesize=10,
                     importance=T)
#Plotando nivel de importancia
varImpPlot(modelo)



```

Com o nível de importancia das variaveis, podemos saber quais variaveis tem o maior impacto para prever a variável valor do aluguel

## Criando modelo de regressão linear

Primeiramento separamos o nosso conjunto de dados em um dataset de treino e outro de teste, 70% dos dados serão utilizados para treinar o modelo e 30% para avaliar a eficácia do mesmo.

```{r}
set.seed(00)
split<- createDataPartition(df$cidade,p=0.7, list=F)
dados_treino= df[split,]
dados_teste=df[-split,]

```

Iremos agora criar o modelo de regressão utilizando apenas as variaveis mais importantes encontradas no feacture select.Seram retirados as variáveis Animal e mobilia por não terem uma significancia para nossa previsão.
```{r}
dados_treino$andar=NULL
dados_teste$andar=NULL
modeloLinear<- train(valor_aluguel~.-total_pago -animal
                     -Furniture,data=dados_treino, method="lm")
```

Com o modelo criado será utilizado os dados de testes para avaliar a precisão do modelo criado
```{r}
previsao<- predict(modeloLinear,dados_teste)
#Dataframe com valor real e valor previsto
score= as.data.frame(cbind(dados_teste$valor_aluguel,previsao))
names(score)=c("Valor_real","Valor_previsto")
head(score)
```

## Avaliando modelo 

Analisando o modelo criado, conseguimos um R-squared de 0.98 indicando que a reta criada se adaptou bem aos dados apresentados. 
```{r warning=FALSE}
summary(modeloLinear)
```

Agora com base nos valores previstos a partir da base de testes, iremos analisar os residuos gerados com base no valor real do aluguel e no valor que o modelo previu.

Avaliando os residuos é possível notar que criamos um bom modelo pelo fato do mesmo seguir uma normal tendendo a zero. Embora possuimos alguns valores que representam os residuos dos pontos fora da curva.

```{r Analisando residuos, warning=FALSE}
residuo<- function(real,previsto){
  return(previsto-real)
}
residu=as.data.frame(mapply( residuo,score$Valor_real,score$Valor_previsto))
names(residu)="residuo"

ggplot(residu,aes(x=residuo), with=600) +geom_histogram(bins=50)+
  xlim(c(-2500,2500)) + theme_classic() + labs(title = "Resíduos entre valores reais e previstos")

```

Em seguida foi plotado os valores reais e previstos em um gráfico para visualizar a diferença entre os valores. É possível notar que os valores ficaram bem proximos um dos outros, mostrando que a análise foi bastante eficaz. Foi mostrado apenas os 500 primeiros valores previstos por motivos de melhor visualização.

```{r plotando valores previstos e reais, warning=FALSE}
score$index=c(1:3205)
ggplotly(ggplot(data=score, aes(x=index,y=Valor_real))+ geom_line(color="red") + labs(title = "Valores Previstos e reais") +geom_line(data=score, aes(x=index, y=Valor_previsto)) +theme_classic() + xlim(c(0,500)))

```


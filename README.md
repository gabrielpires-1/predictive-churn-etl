# Predictive Churn ETL - Projeto da cadeira Big data

Pipeline ETL para ingestão, tratamento e agregação de dados de churn de clientes de telecomunicações, com organização em camadas analíticas (`bronze`, `silver` e `gold`) para apoiar análises exploratórias, indicadores de risco e futuras iniciativas de modelagem preditiva.

## Descrição do projeto

Este projeto foi estruturado para processar dados de cancelamento de clientes de forma incremental e organizada, seguindo uma abordagem em camadas:

- `bronze`: armazenamento da base original ingerida.
- `silver`: dados tratados e padronizados para consumo analítico.
- `gold`: saídas agregadas e visões de negócio, como features do customer, resumo de churn e referência do churn.

O fluxo foi desenvolvido em notebooks para facilitar inspeção, transformação e evolução do processo ao longo das etapas do pipeline.

## Fonte dos dados

Os dados utilizados neste projeto têm como origem o dataset público do Kaggle:

- Dataset: `blastchar/telco-customer-churn`
- Plataforma: Kaggle
- Link: https://www.kaggle.com/datasets/blastchar/telco-customer-churn

## Ferramentas utilizadas

- Python
- JupyterLab
- Pandas
- PyArrow
- KaggleHub
- ipywidgets
- Matplotlib
- Seaborn
  

## Estrutura do repositório

```text
predictive-churn-etl/
|-- bronze/
|   `-- telco_churn_bronze.parquet
|-- silver/
|   `-- telco_churn_silver.parquet
|-- gold/
|   |-- customer_features.parquet
|   |-- churn_reference.parquet
|   `-- churn_summary.parquet
|-- clustering/
|   |-- customer_clusters.parquet
|   |-- cluster_profiles.parquet
|   |-- churn_por_cluster.png
|   |-- cluster_feature_importance.png
|   |-- cluster_radar_chart.png
|   |-- elbow_silhouette.png
|   |-- pca_clusters_2d.png
|   `-- perfil_clusters_heatmap.png
|-- src/
|   |-- 01_ingestion.ipynb
|   |-- 02_transformation.ipynb
|   |-- 03_gold_aggregation.ipynb
|   `-- 04_clustering.ipynb
|-- documentacao/
|   |-- catalogo-de-artefatos.md
|   |-- pipeline-diagram.mmd
|   `-- README.md
|-- dados/
|   `-- README.md
|-- LICENSE
|-- requirements.txt
`-- README.md
```

## Objetivo

Construir uma base confiável para análise de churn, permitindo a:

- rastreabilidade entre os estágios de processamento;
- padronização dos dados para consumo analítico;
- geração de visões consolidadas para suporte à tomada de decisão.

## Resultados da camada Gold

Depois de tratar e consolidar tudo, a base final ficou com 7.032 clientes, e desses cerca de 1.869 acabaram cancelando o serviço. Isso dá uma taxa geral de churn em torno de 26,6%, ou seja, mais ou menos um a cada quatro clientes deixou a operadora. Já é um número alto, mas o mais interessante aparece quando a gente separa esses clientes por tipo de contrato e por quanto tempo eles estão na base.

O padrão que salta aos olhos é o peso do contrato. Quem está no plano mês a mês cancela muito mais do que quem assinou um contrato mais longo, e essa diferença é gritante. No grupo de clientes mais novos, com até um ano de casa e no plano mês a mês, a taxa de cancelamento passa de 50% — praticamente metade some. Esse é de longe o grupo mais arriscado e também um dos maiores em volume, então é onde mora boa parte do problema. Do outro lado, quem fechou contrato de dois anos quase não cancela: nas várias faixas de tempo o churn fica entre zero e pouco mais de 3%.

O tempo de casa também ajuda, mas dentro de cada tipo de contrato. Mesmo no plano mês a mês, conforme o cliente vai ficando mais tempo a tendência de sair vai caindo — começa acima dos 50% nos primeiros meses e desce para perto de 26% entre os clientes mais antigos. Ainda assim, mesmo o cliente "veterano" no mês a mês cancela mais do que um cliente novo já preso num contrato longo, o que reforça que o tipo de plano é o fator que mais pesa.

Vale notar também que os grupos com mais fibra óptica costumam ser justamente os de cobrança mensal mais alta e, no plano mês a mês, os de maior cancelamento — sinal de que o cliente que paga mais é também o mais sensível e o que mais precisa de atenção. Por outro lado, a maior concentração de clientes fiéis está nos contratos de dois anos com bastante tempo de casa: é o maior grupo de toda a base e, ao mesmo tempo, um dos que menos cancelam.

No fim das contas, a leitura é bem direta: o cliente mais perigoso é o recém-chegado no mês a mês pagando uma conta alta, e o caminho mais óbvio para segurar gente é puxar esse pessoal para contratos mais longos.

### Agrupando os clientes por perfil

Além desse corte por contrato e tempo de casa, a gente foi um passo além: usou um algoritmo de agrupamento (K-Means) para deixar os próprios dados juntarem os clientes por semelhança de perfil, sem dizer pra ele nada sobre quem cancelou ou não. Saíram cinco grupos bem diferentes entre si — e o mais interessante é que a taxa de churn muda muito de um pro outro, indo de uns 5% num grupo bem fiel até quase 56% no grupo mais arriscado. São esses grupos que sustentam as respostas das perguntas mais abaixo.

| Grupo | Clientes | Churn | Como é esse cliente |
|---|---|---|---|
| 3 | 1.130 | 5,4% | DSL, contrato longo, muita gente com família e bastante serviço de suporte |
| 1 | 1.523 | 7,4% | clientes sem internet, conta baixa (~R$21) |
| 0 | 1.206 | 20,3% | fibra, contrato longo e conta alta (~R$103) |
| 2 | 1.288 | 30,9% | DSL no mês a mês, maioria morando sozinha |
| 4 | 1.885 | 55,8% | fibra, mês a mês, conta alta, pouco suporte e mais idosos |

### As perguntas que a gente respondeu

Com os grupos em mãos, conseguimos responder quatro perguntas que guiaram o projeto.

**1. Os grupos que mais cancelam concentram clientes idosos? A idade muda o jogo do churn?**

Existe uma relação, sim, mas a idade sozinha não conta a história toda. O grupo que mais cancela (quase 56%) é também o que tem mais idosos — perto de 30% deles, contra menos de 9% nos grupos mais fiéis. Só que, olhando de perto, o que realmente derruba esse grupo é o pacote inteiro: é quase todo mundo no mês a mês, com fibra, pagando por electronic check, com pouco suporte e pouco tempo de casa. Ou seja, o idoso parece ser mais sensível a esse cenário de pouca estabilidade e pouco apoio — quando ele tem contrato mais firme e suporte por trás, a chance de sair cai bastante.

**2. Ter cônjuge e/ou dependentes empurra o cliente para os grupos de menor churn?**

Sim, e essa foi uma das relações mais fortes que apareceram. Quem tem cônjuge e dependentes cancela em torno de 14%; já quem mora sozinho e sem ninguém dependente pula pra 34%. Isso se reflete direto nos grupos: os mais fiéis são justamente os mais "familiares" — no grupo de menor churn, três em cada quatro clientes têm família. A correlação entre "ter família" e a taxa de cancelamento ficou em −0,75, que é considerada alta: quanto mais família, menos churn.

**3. O tipo de internet contratada diferencia os grupos e mexe no churn?**

Sim — e esse acabou sendo o fator que mais separou os grupos, a ponto de cada um ser quase "puro" num tipo de internet. O churn sobe junto com o tipo (e o preço) do serviço: quem não tem internet cancela só 7,4%, quem tem DSL 19% e quem tem fibra 41,9%. Quer dizer, o cliente de fibra cancela quase 6× mais que quem não tem internet, e é também quem paga mais caro (média de R$91 na fibra, contra R$58 na DSL e R$21 sem internet). Mas tem um porém importante: o tipo de internet sozinho não decide nada — a mesma DSL aparece tanto no grupo mais fiel quanto num de churn alto. E um detalhe curioso: dentro de cada tipo, quem cancela até paga um pouco menos que quem fica. Então o problema da fibra parece estar mais na percepção de custo-benefício da categoria do que no valor exato da conta.

**4. Existe algum grupo no mês a mês que mesmo assim segura o churn lá embaixo?**

Pronto, não existe — nenhum grupo junta "muito contrato mensal" com "churn baixo". O contrato é disparado o que mais separa quem fica de quem sai: contrato de dois anos cancela só 2,8%, de um ano 11,3%, e o mês a mês 42,7%. Não por acaso, os dois grupos com mais gente no mês a mês são justamente os que mais cancelam. Ainda assim, dentro do próprio mês a mês dá pra reduzir bastante o risco:

- **Tempo de casa**: nos primeiros meses o churn beira os 51%, mas vai caindo e chega a 26% entre os clientes mais antigos.
- **Suporte técnico**: com suporte ativo cai pra 30,7% (contra 45,2% sem), e quem tem três ou mais serviços de suporte cai pra 23,4%.
- **Juntando tudo**: um cliente mês a mês com mais de quatro anos de casa e suporte ativo cancela só 10,9% — quase no nível de quem assinou contrato anual.

No fim, a real é que não dá pra "consertar" o contrato mensal, mas dá pra deixar ele bem menos arriscado com tempo de casa e suporte por trás.


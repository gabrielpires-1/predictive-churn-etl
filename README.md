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


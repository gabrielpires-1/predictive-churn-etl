# Predictive Churn ETL - Projeto da cadeira Big data

Pipeline ETL para ingestão, tratamento e agregação de dados de churn de clientes de telecomunicações, com organização em camadas analíticas (`bronze`, `silver` e `gold`) para apoiar análises exploratórias, indicadores de risco e futuras iniciativas de modelagem preditiva.

## Descrição do projeto

Este projeto foi estruturado para processar dados de cancelamento de clientes de forma incremental e organizada, seguindo uma abordagem em camadas:

- `bronze`: armazenamento da base original ingerida.
- `silver`: dados tratados e padronizados para consumo analítico.
- `gold`: saídas agregadas e visões de negócio, como resumo de churn e matriz de risco.

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

## Estrutura do repositório

```text
predictive-churn-etl/
|-- bronze/
|-- silver/
|-- gold/
|-- src/
|   |-- 01_ingestion.ipynb
|   |-- 02_transformation.ipynb
|   `-- 03_gold_aggregation.ipynb
|-- requirements.txt
`-- README.md
```

## Objetivo

Construir uma base confiável para análise de churn, permitindo:

- rastreabilidade entre os estágios de processamento;
- padronização dos dados para consumo analítico;
- geração de visões consolidadas para suporte à tomada de decisão.

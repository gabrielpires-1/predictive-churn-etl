# Catalogo de Artefatos

## Fonte de dados

### Dataset de origem

- Nome: `blastchar/telco-customer-churn`
- Plataforma: Kaggle
- Arquivo principal: `WA_Fn-UseC_-Telco-Customer-Churn.csv`

## Camada Bronze

### Arquivo

- `bronze/telco_churn_bronze.parquet`

### Descrição

Representa a persistencia inicial dos dados apos a ingestao da fonte externa, com foco em rastreabilidade e preservacao do conteudo bruto.

## Camada Silver

### Arquivo

- `silver/telco_churn_silver.parquet`

### Descrição

Contem os dados tratados, com ajustes de tipos, limpeza, validações e preparacao para consumo analitico.

## Camada Gold

### Arquivos

- `gold/churn_summary.parquet `
- `gold/risk_matrix.parquet`
### Descricao

Agrupa os produtos finais do pipeline, orientados a leitura de negocio e apoio a visualizacoes ou modelos futuros.

## Notebooks do projeto

### `src/01_ingestion.ipynb`

Responsavel por baixar/carregar o dataset e gravar a camada Bronze.

### `src/02_transformation.ipynb`

Responsavel por limpar, validar e transformar os dados da Bronze para a Silver.

### `src/03_gold_aggregation.ipynb`

Responsável por gerar agregações e estruturas analíticas da camada Gold.

## Possíveis extensoes futuras

- tabela de features para machine learning;
- dashboards de churn;
- métricas de qualidade de dados por camada;
- automaçao de pipeline (fora do ambiente de notebook).

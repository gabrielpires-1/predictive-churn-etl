# Predictive Churn ETL — Análise de Cancelamento de Clientes em Telecomunicações

> Projeto da cadeira de **Big Data**.
> Pipeline de dados em camadas (`bronze` → `silver` → `gold`) com segmentação de clientes por aprendizado não supervisionado (K-Means) para entender e antecipar o churn em uma operadora de telecom.

## Sumário

1. [Introdução](#1-introdução)
2. [Motivação](#2-motivação)
3. [Objetivo do Projeto](#3-objetivo-do-projeto)
4. [Metodologia (Pipeline de Dados)](#4-metodologia-pipeline-de-dados)
5. [Resultados e Visualizações](#5-resultados-e-visualizações)
6. [Conclusões](#6-conclusões)
7. [Estrutura do Repositório](#7-estrutura-do-repositório)
8. [Como Executar](#8-como-executar)
9. [Fonte dos Dados e Licença](#9-fonte-dos-dados-e-licença)

---

## 1. Introdução

**Churn** é o termo usado para descrever o cancelamento de clientes — quando alguém deixa de usar um serviço e vai embora. Em empresas de telecomunicações, onde a receita é recorrente (o cliente paga mês a mês), o churn é uma das métricas mais sensíveis do negócio: cada cliente que cancela representa receita futura perdida e, normalmente, um custo de aquisição que não se pagou por completo.

O problema central que este projeto investiga é simples de enunciar e difícil de responder: **quem cancela, por quê, e o que diferencia o cliente que fica do cliente que sai?** Em vez de tratar o churn como um número único e agregado, o projeto organiza os dados de forma incremental e rastreável e, a partir deles, segmenta a base de clientes em perfis para encontrar onde o risco de cancelamento realmente se concentra.

O trabalho usa a base pública **Telco Customer Churn**, que descreve cerca de 7 mil clientes de uma operadora fictícia com informações demográficas, de serviços contratados, de contrato e de cobrança, além da informação de quem cancelou.

## 2. Motivação

A escolha do tema se justifica por três motivos:

- **Relevância de negócio.** Reter um cliente costuma ser várias vezes mais barato do que conquistar um novo. Entender *quais* clientes estão em risco — e *por quê* — permite agir antes do cancelamento, com ações direcionadas (oferta de contrato mais longo, suporte técnico, ajuste de plano) em vez de campanhas genéricas e caras.
- **Aderência ao conteúdo da cadeira.** O problema é um caso clássico e completo de engenharia e análise de dados: exige ingestão de uma fonte externa, limpeza e padronização de dados sujos, criação de features, agregações analíticas e, por fim, um modelo de aprendizado de máquina. É o ciclo inteiro de um pipeline de dados, ponta a ponta.
- **Reprodutibilidade.** O dataset é público e bem conhecido, o que torna o projeto fácil de auditar, reproduzir e comparar com outras abordagens.

## 3. Objetivo do Projeto

O objetivo geral é **construir uma base confiável e rastreável para análise de churn** e, sobre ela, **segmentar os clientes por perfil** para responder a perguntas concretas de negócio.

Desdobrando em objetivos específicos:

- Implementar um **pipeline ETL em camadas** (arquitetura *medallion*: bronze, silver, gold), garantindo rastreabilidade entre cada estágio de processamento.
- **Padronizar e enriquecer** os dados brutos, criando features que descrevem o perfil de consumo, contrato, pagamento e estrutura familiar de cada cliente.
- **Agrupar os clientes por semelhança** com um algoritmo não supervisionado (K-Means), *sem* informar ao modelo quem cancelou, e só então cruzar os grupos com o churn real.
- Usar essa segmentação para **responder a quatro perguntas de negócio** que guiaram o projeto (idade, estrutura familiar, tipo de internet e contrato mensal).
- Deixar a base preparada para **futuras iniciativas de modelagem preditiva** e dashboards.

## 4. Metodologia (Pipeline de Dados)

### 4.1 Arquitetura da solução

O pipeline segue a **arquitetura em camadas (medallion)**, em que os dados avançam por estágios de qualidade crescente. Cada etapa é um notebook independente, lê o artefato da etapa anterior e grava o seu próprio artefato em **Parquet** (formato colunar que preserva tipos e é eficiente para leitura analítica).

```
Kaggle (CSV)  ──►  BRONZE  ──►  SILVER  ──►  GOLD  ──►  ML / Clustering  ──►  Consumo analítico
                  (bruto)     (tratado)   (features)    (segmentos)         (insights / dashboards)
```

O diagrama completo do fluxo está em [`documentacao/pipeline-diagram.mmd`](documentacao/pipeline-diagram.mmd) (visualizável em [mermaid.live](https://mermaid.live)). O catálogo descritivo de cada artefato gerado está em [`documentacao/catalogo-de-artefatos.md`](documentacao/catalogo-de-artefatos.md).

### 4.2 Fontes

| Item | Descrição |
|---|---|
| Dataset | `blastchar/telco-customer-churn` |
| Plataforma | Kaggle |
| Arquivo | `WA_Fn-UseC_-Telco-Customer-Churn.csv` |
| Volume | ~7.043 clientes × 21 colunas |
| Conteúdo | dados demográficos, serviços contratados, tipo de contrato, forma de pagamento, cobranças e flag de churn |
| Link | https://www.kaggle.com/datasets/blastchar/telco-customer-churn |

### 4.3 Ingestão — Camada Bronze

**Notebook:** [`src/01_ingestion.ipynb`](src/01_ingestion.ipynb)

A ingestão baixa o dataset diretamente do Kaggle via **`kagglehub`** (já entregando um DataFrame Pandas) e persiste o conteúdo **bruto, sem qualquer transformação**, em `bronze/telco_churn_bronze.parquet`. O objetivo desta camada é apenas garantir rastreabilidade e ter uma cópia fiel da origem para reprocessar quando necessário.

### 4.4 Transformação — Camada Silver

**Notebook:** [`src/02_transformation.ipynb`](src/02_transformation.ipynb)

É a etapa mais densa do pipeline. Lê a camada Bronze e produz um dataset limpo, tipado e enriquecido em `silver/telco_churn_silver.parquet`:

1. **Auditoria de qualidade** — inspeção de tipos, nulos, cardinalidade e valores possíveis de cada coluna.
2. **Correção de tipos** — `TotalCharges` veio como texto por conter 11 linhas em branco (clientes com `tenure = 0`); foi convertida para numérica.
3. **Remoção de nulos** — as 11 linhas inconsistentes (`tenure = 0`) foram descartadas, deixando a base com **7.032 clientes**.
4. **Padronização binária** — colunas `Yes/No` e `Male/Female` convertidas para `0/1`.
5. **Colapso de serviços** — variáveis de serviço (suporte, streaming, etc.) simplificadas para `1` (tem) / `0` (não tem).
6. **Features derivadas**, entre elas:
   - `tenure_group` — faixas de tempo de casa (0-12, 13-24, 25-48, 49-72 meses);
   - `charges_per_month_ratio` — gasto médio mensal ao longo da permanência;
   - `service_count`, `support_services_count`, `streaming_services_count` — profundidade de uso do pacote;
   - flags de perfil de internet (`sem internet`, `DSL`, `fibra`);
   - variáveis de contrato (`contract_term_months`, `is_month_to_month`, `is_long_term_contract`);
   - perfil de pagamento (`payment_automatic`, `payment_electronic_check`, ...);
   - estrutura familiar (`family_size`, `has_family`, `is_single_no_dependents`, `household_profile`);
   - faixas de cobrança (`monthly_charge_band`).
7. **Normalização Min-Max** — as colunas contínuas mais úteis foram escaladas para o intervalo `[0, 1]` (sufixo `_norm`), preservando as versões originais. Isso é essencial porque o K-Means é baseado em distância: sem normalizar, variáveis com escalas grandes (como `TotalCharges`) dominariam o agrupamento.
8. **Validação do output** — checagem de ausência de nulos e do schema final.

### 4.5 Carregamento — Camada Gold

**Notebook:** [`src/03_gold_aggregation.ipynb`](src/03_gold_aggregation.ipynb)

Consolida a Silver em três produtos analíticos prontos para consumo:

| Artefato | Conteúdo |
|---|---|
| `gold/customer_features.parquet` | **27 features numéricas** selecionadas para a clusterização (apenas numéricas, sem nulos e sem o alvo) |
| `gold/churn_reference.parquet` | mapa `customerID → Churn` — separa o alvo das features para cruzar cada cliente com o cluster e calcular a taxa de churn do grupo |
| `gold/churn_summary.parquet` | resumo descritivo por `tenure_group` × `Contract` (taxa de churn, volume, cobrança média, proporção de fibra, etc.) |

A separação entre `customer_features` e `churn_reference` é deliberada: o modelo de agrupamento **não enxerga** quem cancelou; o churn só é reintroduzido *depois*, para validar os segmentos.

### 4.6 Camada Analítica / ML — Clusterização

**Notebook:** [`src/04_clustering.ipynb`](src/04_clustering.ipynb)

Etapa de aprendizado **não supervisionado** que descobre perfis de cliente a partir das 27 features:

- **Definição do K** — método do cotovelo (*elbow*) combinado com *silhouette score* ao longo de `k = 2..10`.
- **Treinamento** — **K-Means** com `K = 5` (`n_init = 20`, `random_state = 42` para reprodutibilidade).
- **Validação quantitativa** — *Silhouette*, *Calinski-Harabasz* e *Davies-Bouldin* para medir a qualidade da separação.
- **Validação de negócio** — cruzamento dos clusters com `churn_reference` para medir a taxa de churn de cada grupo.
- **Interpretação** — análise dos centróides, projeção **PCA 2D** para visualização e **Random Forest** para estimar a importância das features que mais distinguem os clusters.

### 4.7 Destino e consumo

Os artefatos finais (`gold/` e `clustering/`) servem de base para **exploração analítica**, **visualizações** e futuros **dashboards** e **modelos preditivos**. A pasta [`dados/`](dados/) reserva espaço para amostras pequenas; arquivos grandes não são versionados.

### Tecnologias utilizadas

| Categoria | Ferramentas |
|---|---|
| Linguagem | Python |
| Ambiente | JupyterLab |
| Manipulação de dados | Pandas, NumPy |
| Armazenamento | PyArrow (Parquet) |
| Ingestão | KaggleHub |
| Machine Learning | scikit-learn (K-Means, PCA, Random Forest, métricas) |
| Visualização | Matplotlib, Seaborn |
| Interface | ipywidgets |

> As versões exatas estão fixadas em [`requirements.txt`](requirements.txt).

## 5. Resultados e Visualizações

### 5.1 Panorama geral

Depois de tratar e consolidar tudo, a base final ficou com **7.032 clientes**, dos quais cerca de **1.869 cancelaram** o serviço — uma taxa geral de churn em torno de **26,6%**, ou seja, mais ou menos um a cada quatro clientes deixou a operadora. É um número alto, mas o mais interessante aparece quando separamos os clientes por **tipo de contrato** e por **tempo de casa**.

O padrão que salta aos olhos é o peso do contrato. Quem está no plano mês a mês cancela muito mais do que quem assinou um contrato mais longo, e a diferença é gritante. No grupo de clientes mais novos (até um ano de casa, plano mês a mês), a taxa de cancelamento passa de **50%** — praticamente metade some. É de longe o grupo mais arriscado e também um dos maiores em volume. Do outro lado, quem fechou contrato de dois anos quase não cancela: nas várias faixas de tempo o churn fica entre **0% e ~3%**.

O tempo de casa também ajuda, mas dentro de cada tipo de contrato: mesmo no mês a mês, conforme o cliente vai ficando, a tendência de sair cai — começa acima de 50% nos primeiros meses e desce para perto de 26% entre os mais antigos. Ainda assim, o cliente "veterano" do mês a mês cancela mais do que um cliente novo já preso num contrato longo, o que reforça que **o tipo de plano é o fator que mais pesa**.

Vale notar que os grupos com mais fibra óptica são justamente os de cobrança mensal mais alta e, no mês a mês, os de maior cancelamento — sinal de que o cliente que paga mais é também o mais sensível. A maior concentração de clientes fiéis está nos contratos de dois anos com bastante tempo de casa.

### 5.2 Segmentação por perfil (K-Means)

Indo além do corte por contrato e tempo de casa, usamos o **K-Means** para deixar os próprios dados juntarem os clientes por semelhança de perfil — sem dizer nada ao modelo sobre quem cancelou. Saíram **cinco grupos** bem diferentes entre si, e a taxa de churn varia muito de um para o outro, de ~5% no grupo mais fiel até quase 56% no mais arriscado.

| Grupo | Clientes | Churn | Como é esse cliente |
|---|---|---|---|
| 3 | 1.130 | 5,4% | DSL, contrato longo, muita gente com família e bastante serviço de suporte |
| 1 | 1.523 | 7,4% | clientes sem internet, conta baixa (~R$21) |
| 0 | 1.206 | 20,3% | fibra, contrato longo e conta alta (~R$103) |
| 2 | 1.288 | 30,9% | DSL no mês a mês, maioria morando sozinha |
| 4 | 1.885 | 55,8% | fibra, mês a mês, conta alta, pouco suporte e mais idosos |

**Visualizações geradas** (pasta [`clustering/`](clustering/)):

| Visualização | Arquivo |
|---|---|
| Escolha do K (elbow + silhouette) | [`elbow_silhouette.png`](clustering/elbow_silhouette.png) |
| Projeção 2D dos clusters (PCA) | [`pca_clusters_2d.png`](clustering/pca_clusters_2d.png) |
| Churn por cluster | [`churn_por_cluster.png`](clustering/churn_por_cluster.png) |
| Importância das features (Random Forest) | [`cluster_feature_importance.png`](clustering/cluster_feature_importance.png) |
| Radar dos perfis de cluster | [`cluster_radar_chart.png`](clustering/cluster_radar_chart.png) |
| Heatmap de perfil dos clusters | [`perfil_clusters_heatmap.png`](clustering/perfil_clusters_heatmap.png) |

### 5.3 As perguntas de negócio respondidas

Com os grupos em mãos, conseguimos responder às quatro perguntas que guiaram o projeto.

**1. Os grupos que mais cancelam concentram clientes idosos? A idade muda o jogo do churn?**

Existe uma relação, sim, mas a idade sozinha não conta a história toda. O grupo que mais cancela (quase 56%) é também o que tem mais idosos — perto de **30%** deles, contra menos de 9% nos grupos mais fiéis. Só que, olhando de perto, o que realmente derruba esse grupo é o pacote inteiro: é quase todo mundo no mês a mês, com fibra, pagando por *electronic check*, com pouco suporte e pouco tempo de casa. Ou seja, o idoso parece ser mais sensível a esse cenário de pouca estabilidade e pouco apoio — quando tem contrato mais firme e suporte por trás, a chance de sair cai bastante.

**2. Ter cônjuge e/ou dependentes empurra o cliente para os grupos de menor churn?**

Sim, e essa foi uma das relações mais fortes que apareceram. Quem tem cônjuge e dependentes cancela em torno de **14%**; já quem mora sozinho e sem dependentes pula para **34%**. Isso se reflete direto nos grupos: os mais fiéis são justamente os mais "familiares" — no grupo de menor churn, três em cada quatro clientes têm família. A correlação entre "ter família" e a taxa de cancelamento ficou em **−0,75**, considerada alta: quanto mais família, menos churn.

**3. O tipo de internet contratada diferencia os grupos e mexe no churn?**

Sim — e esse acabou sendo o fator que mais separou os grupos, a ponto de cada um ser quase "puro" num tipo de internet. O churn sobe junto com o tipo (e o preço) do serviço: quem **não tem internet** cancela só **7,4%**, **DSL** cancela **19,0%** e **fibra** cancela **41,9%**. O cliente de fibra cancela quase **6× mais** que quem não tem internet, e é também quem paga mais caro (média de ~R$91 na fibra, contra ~R$58 na DSL e ~R$21 sem internet). Mas há um porém: o tipo de internet sozinho não decide nada — a mesma DSL aparece tanto no grupo mais fiel quanto num de churn alto. E um detalhe curioso: dentro de cada tipo, quem cancela até paga um pouco *menos* que quem fica. Então o problema da fibra parece estar mais na percepção de custo-benefício da categoria do que no valor exato da conta.

**4. Existe algum grupo no mês a mês que mesmo assim segura o churn lá embaixo?**

Não — nenhum grupo junta "muito contrato mensal" com "churn baixo". O contrato é disparado o que mais separa quem fica de quem sai: contrato de dois anos cancela só **2,8%**, de um ano **11,3%**, e o mês a mês **42,7%**. Não por acaso, os dois grupos com mais gente no mês a mês são justamente os que mais cancelam. Ainda assim, dentro do próprio mês a mês dá para reduzir bastante o risco:

- **Tempo de casa:** nos primeiros meses o churn beira os 51%, mas cai para 26% entre os clientes mais antigos.
- **Suporte técnico:** com suporte ativo cai para 30,7% (contra 45,2% sem); com três ou mais serviços de suporte, cai para 23,4%.
- **Juntando tudo:** um cliente mês a mês com mais de quatro anos de casa e suporte ativo cancela só **10,9%** — quase no nível de quem assinou contrato anual.

A leitura final é direta: não dá para "consertar" o contrato mensal, mas dá para deixá-lo bem menos arriscado com tempo de casa e suporte por trás.

## 6. Conclusões

### 6.1 Análise crítica dos resultados

O projeto confirma que **o churn em telecom não é aleatório nem explicado por uma variável só** — ele é a soma de um *pacote* de condições. O cliente mais perigoso tem um perfil bem definido: **recém-chegado, no plano mês a mês, com fibra, pagando conta alta, com pouco suporte e morando sozinho**. O cliente mais fiel é quase o espelho disso: contrato longo, com família e com serviços de suporte contratados.

O ponto mais valioso da análise é que a segmentação não supervisionada (K-Means) **chegou sozinha às mesmas conclusões** dos cortes manuais por contrato e tempo de casa — o que dá confiança nos achados. E a clusterização foi além ao mostrar que fatores como contrato, tipo de internet e estrutura familiar se combinam em perfis coerentes, em vez de atuarem isoladamente.

Do ponto de vista de negócio, o caminho mais óbvio para reter clientes é **migrar o público de risco para contratos mais longos** e **reforçar suporte e vínculo** — especialmente para o cliente de fibra e para o público idoso, que é mais sensível à instabilidade.

### 6.2 Dificuldades encontradas

- **Dados sujos disfarçados.** `TotalCharges` chegou como texto por causa de 11 linhas em branco (clientes novos com `tenure = 0`); só uma auditoria cuidadosa de tipos revelou o problema. `SeniorCitizen` já vinha como 0/1 enquanto o resto da base usava Yes/No, exigindo tratamento diferente.
- **Escalas heterogêneas.** Como o K-Means é baseado em distância, foi preciso normalizar as variáveis contínuas (Min-Max) para que nenhuma feature dominasse o agrupamento.
- **Escolha do número de clusters.** O K ideal não é óbvio; foi necessário combinar o método do cotovelo com o *silhouette score* e validar os grupos pelo significado de negócio, não só pela métrica.
- **Correlação não é causalidade.** A relação entre idade e churn, por exemplo, exigiu cuidado interpretativo para não atribuir à idade o que era efeito do contrato e do tipo de serviço.

### 6.3 Trabalhos futuros

- **Modelo preditivo supervisionado** (ex.: regressão logística / gradient boosting) para estimar a probabilidade de churn por cliente, aproveitando a `customer_features` já pronta.
- **Dashboards de churn** interativos sobre a camada Gold.
- **Métricas de qualidade de dados** por camada (testes automatizados de schema e nulos).
- **Orquestração do pipeline** fora do ambiente de notebook (ex.: Airflow / scripts parametrizados) para execução automatizada e agendada.

## 7. Estrutura do Repositório

```text
predictive-churn-etl/
├── src/                          # Código-fonte: notebooks do pipeline e da análise
│   ├── 01_ingestion.ipynb        #   Etapa 1 — Ingestão (Bronze)
│   ├── 02_transformation.ipynb   #   Etapa 2 — Transformação (Silver)
│   ├── 03_gold_aggregation.ipynb #   Etapa 3 — Agregação (Gold)
│   └── 04_clustering.ipynb       #   Etapa 4 — Clusterização (ML)
├── bronze/                       # Camada Bronze — dados brutos
│   └── telco_churn_bronze.parquet
├── silver/                       # Camada Silver — dados tratados
│   └── telco_churn_silver.parquet
├── gold/                         # Camada Gold — produtos analíticos
│   ├── customer_features.parquet
│   ├── churn_reference.parquet
│   └── churn_summary.parquet
├── clustering/                   # Saídas do modelo: parquets e visualizações (.png)
│   ├── customer_clusters.parquet
│   ├── cluster_profiles.parquet
│   ├── elbow_silhouette.png
│   ├── pca_clusters_2d.png
│   ├── churn_por_cluster.png
│   ├── cluster_feature_importance.png
│   ├── cluster_radar_chart.png
│   └── perfil_clusters_heatmap.png
├── documentacao/                 # Material de apoio: diagramas e catálogo
│   ├── pipeline-diagram.mmd
│   ├── catalogo-de-artefatos.md
│   └── README.md
├── dados/                        # Amostras pequenas de dados (arquivos grandes não versionados)
│   └── README.md
├── requirements.txt
├── LICENSE
└── README.md
```

> **Observação sobre as pastas.** Por todo o desenvolvimento ter sido feito em Jupyter Notebooks, o código e os notebooks de exploração/análise convivem na pasta `src/` (que cumpre o papel de `/codigo` e de `/notebooks`). Os notebooks `01`–`03` constroem o pipeline ETL e o `04` concentra a exploração analítica e a modelagem.

## 8. Como Executar

```bash
# 1. (Opcional) crie e ative um ambiente virtual
python -m venv .venv && source .venv/bin/activate

# 2. Instale as dependências
pip install -r requirements.txt

# 3. Configure as credenciais do Kaggle (necessário só para a ingestão)
#    Coloque seu kaggle.json em ~/.kaggle/ ou exporte as variáveis:
#    export KAGGLE_USERNAME=...  KAGGLE_KEY=...

# 4. Abra o JupyterLab
jupyter lab
```

Execute os notebooks **na ordem**, pois cada um consome o artefato do anterior:

`01_ingestion` → `02_transformation` → `03_gold_aggregation` → `04_clustering`

## 9. Fonte dos Dados e Licença

- **Dataset:** [Telco Customer Churn](https://www.kaggle.com/datasets/blastchar/telco-customer-churn) — `blastchar/telco-customer-churn` (Kaggle).
- **Licença do projeto:** ver [`LICENSE`](LICENSE).

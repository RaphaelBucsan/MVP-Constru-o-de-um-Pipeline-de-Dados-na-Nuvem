# MVP — Construção de um Pipeline de Dados na Nuvem com Databricks

## O pipeline foi desenvolvido e executado no Databricks. Todo o código-fonte dos notebooks utilizados no pipeline e na análise está disponível neste repositório. As evidências de execução, persistência das tabelas e Workflow estão apresentadas ao longo da documentação.

## 1. Contexto de Negócio e Perguntas

Este projeto tem como objetivo construir um pipeline de dados de ponta a ponta em ambiente de nuvem utilizando a base pública Olist, estruturando os dados desde a ingestão bruta até a disponibilização de informações analíticas prontas para consumo.

A proposta é organizar um fluxo reproduzível capaz de responder perguntas relacionadas a vendas, comportamento de clientes, categorias de produtos e desempenho logístico.

Perguntas de negócio utilizadas como orientação:

1. Como as vendas evoluem ao longo do tempo?
2. Quais categorias concentram maior volume de pedidos, itens e receita?
3. Qual é o comportamento de recompra dos clientes?
4. Clientes recorrentes apresentam comportamento financeiro diferente dos clientes de compra única?
5. Qual é o desempenho logístico global dos pedidos?
6. Existem diferenças relevantes de desempenho logístico entre as UFs?
7. Quais sellers aparecem com maior frequência em pedidos atrasados?
8. Quais categorias apresentam maior associação com atrasos?
9. Características físicas dos produtos, como peso e volume, apresentam associação com o desempenho logístico?

Os dados utilizados representam operações de e-commerce e incluem informações de clientes, pedidos, itens, pagamentos, produtos, vendedores, avaliações, geolocalização e tradução de categorias.

Observação sobre licença:
A documentação final do repositório deve informar a licença do conjunto de dados Olist conforme a fonte pública utilizada na coleta. Recomenda-se registrar no README a URL original da base e a licença apresentada na página oficial do dataset.

[INSERIR SCREENSHOT 01 — visão geral da fonte de dados ou estrutura dos arquivos brutos]


## 2. Coleta e Carga dos Dados

A ingestão foi realizada no Databricks a partir de arquivos CSV da base Olist.

Foram considerados os seguintes conjuntos de dados:

- olist_customers_dataset.csv
- olist_geolocation_dataset.csv
- olist_order_items_dataset.csv
- olist_order_payments_dataset.csv
- olist_order_reviews_dataset.csv
- olist_orders_dataset.csv
- olist_products_dataset.csv
- olist_sellers_dataset.csv
- product_category_name_translation.csv

O notebook responsável por essa etapa é:

01_ingestao_bronze.ipynb

Na camada Bronze, o objetivo foi preservar o conteúdo recebido da fonte, evitando aplicação de regras de negócio nessa etapa.

Foram adicionados metadados técnicos de rastreabilidade:

- _ingestion_timestamp
- _source_file

As tabelas foram persistidas no ambiente Databricks para permitir rastreabilidade, reprocessamento e evolução posterior do pipeline.

[INSERIR SCREENSHOT 02 — tabelas Bronze persistidas no Databricks]


## 3. Arquitetura de Dados

O projeto utiliza arquitetura Medallion, organizada em três camadas principais:

Bronze
Dados brutos ingeridos com mínima intervenção.

Silver
Dados tratados, padronizados e enriquecidos.

Gold
Dados modelados para consumo analítico, incluindo dimensões, fatos e agregados.

Fluxo lógico:

Bronze
  ↓
Qualidade Bronze
  ↓
Silver
  ↓
Gold
  ↓
Agregações Gold
  ↓
Quality Gate
  ↓
Análise

O pipeline principal foi separado em notebooks com responsabilidades distintas:

01_ingestao_bronze.ipynb
02_qualidade_dados_bronze.ipynb
03_transformacao_silver.ipynb
04_modelagem_gold.ipynb
05_agregacoes_gold.ipynb
06_validacao_gold.ipynb

O notebook 07_analise_gold.ipynb foi mantido fora do Workflow ETL, pois representa a camada de consumo analítico e não a construção do pipeline.

[INSERIR SCREENSHOT 03 — arquitetura do Workflow no Databricks]


## 4. Modelagem e Catálogo de Dados

A modelagem Gold foi estruturada para permitir análises em diferentes granularidades sem misturar fatos incompatíveis.

Principais dimensões:

- dim_customer
- dim_seller
- dim_product
- dim_date
- dim_order

Principais fatos:

- fact_order_items
- fact_payments

Agregados analíticos:

- agg_vendas_mensais
- agg_vendas_categoria
- agg_clientes
- agg_logistica_uf

Granularidades principais:

dim_customer
1 linha por customer_id.

dim_seller
1 linha por seller_id.

dim_product
1 linha por product_id.

dim_date
1 linha por dia.

dim_order
1 linha por order_id.

fact_order_items
1 linha por combinação order_id + order_item_id.

fact_payments
1 linha por combinação order_id + payment_sequential.

A separação entre fact_order_items e fact_payments evita problemas de multiplicação de linhas em relações muitos-para-muitos entre itens e pagamentos.

A dim_order foi utilizada para análises de ciclo logístico no grão correto de pedido e inclui customer_id, permitindo relacionamento com dim_customer.

O catálogo completo foi documentado em arquivo separado:

Catalogo_Dados_MVP_Olist.xlsx

O catálogo contém:

- tabela;
- granularidade;
- campo;
- tipo lógico;
- descrição;
- domínio/regra;
- transformação;
- linhagem.

[INSERIR SCREENSHOT 04 — Unity Catalog ou estrutura das tabelas Gold]
[INSERIR SCREENSHOT 05 — exemplo do catálogo de dados]


## 5. Transformações e Regras da Camada Silver

A camada Silver concentra as principais regras de limpeza e padronização.

Exemplos de transformações realizadas:

- padronização de cidades com trim e lower;
- padronização de estados com trim e upper;
- padronização de order_status;
- padronização de payment_type;
- conversão de valores monetários para double;
- ajuste dos nomes product_name_lenght e product_description_lenght;
- tratamento de peso inválido;
- limpeza de comentários vazios;
- consolidação da geolocalização por prefixo de CEP;
- tradução de categorias de produtos;
- criação de indicadores de inconsistência temporal;
- criação de métricas logísticas.

Métricas logísticas derivadas:

delivery_time_days
Diferença entre a data de compra e a data de entrega ao cliente.

delivery_delay_days
Diferença entre a data real de entrega e a data estimada.

flag_late_delivery
Indicador igual a 1 quando delivery_delay_days > 0 e 0 nos demais casos.

Também foram criadas flags para identificar sequências temporais inconsistentes, como aprovação anterior à compra ou entrega anterior à etapa de transporte.


## 6. Pipeline de Dados

O pipeline foi implementado com Databricks Workflows.

Dependências:

01_ingestao_bronze
  ↓
02_qualidade_dados_bronze
  ↓
03_transformacao_silver
  ↓
04_modelagem_gold
  ↓
05_agregacoes_gold
  ↓
06_validacao_gold

O Workflow foi executado de ponta a ponta com sucesso.

Essa organização separa claramente:

- ingestão;
- diagnóstico de qualidade;
- transformação;
- modelagem;
- agregações;
- validação final.

Essa separação também reduz acoplamento entre etapas e facilita manutenção, reprocessamento e identificação de falhas.

[INSERIR SCREENSHOT 06 — Workflow completo com todas as tasks]
[INSERIR SCREENSHOT 07 — execução do Job com status Succeeded]


## 7. Qualidade de Dados

A qualidade dos dados foi avaliada em diferentes momentos do pipeline.

Na camada Bronze foram analisados aspectos como:

- completude;
- unicidade;
- consistência;
- plausibilidade;
- outliers.

As regras de qualidade foram documentadas e utilizadas para orientar as transformações da Silver.

Na Gold foi criado um Quality Gate final para validar:

- unicidade das dimensões;
- unicidade das fatos;
- integridade referencial;
- consistência das regras logísticas;
- ausência de valores monetários negativos;
- consistência entre preço, frete e valor total;
- consistência das chaves de data;
- integridade entre pedido e cliente;
- reconciliação dos agregados.

Antes da refatoração final, o Quality Gate já possuía 21 testes aprovados. Após a inclusão dos agregados e da relação dim_order → dim_customer, o notebook de validação foi atualizado para cobrir a arquitetura final.

A execução final do Workflow foi concluída com sucesso, indicando que nenhuma regra de validação bloqueante falhou.

[INSERIR SCREENSHOT 08 — saída do Quality Gate]
[INSERIR SCREENSHOT 09 — execução da task 06_validacao_gold]


## 8. Análise de Dados

As análises foram realizadas no notebook:

07_analise_gold.ipynb

Esse notebook consome a camada Gold e não faz parte do Workflow de construção do ETL.

### 8.1 Visão comercial

A análise mensal consolidou:

- número de pedidos;
- número de itens;
- valor de produtos;
- valor de frete;
- valor total;
- ticket médio;
- itens por pedido;
- crescimento mensal.

A tabela agg_vendas_mensais foi persistida na Gold para permitir reuso analítico.

[INSERIR SCREENSHOT 10 — evolução mensal de vendas]


### 8.2 Categorias

A análise por categoria permitiu avaliar:

- pedidos;
- itens vendidos;
- valor de produtos;
- frete;
- receita total;
- preço médio;
- participação na receita;
- ranking de receita.

Foram identificados 1.603 itens sem categoria definida, mantidos na análise como condição conhecida da fonte.

[INSERIR SCREENSHOT 11 — ranking de categorias]


### 8.3 Clientes e recorrência

A análise utilizou customer_unique_id como identificador do cliente ao longo do período observado.

Foram identificados:

- 96.096 clientes únicos;
- 93.099 clientes com uma única compra;
- 2.997 clientes recorrentes;
- taxa de recorrência de 3,12%.

Os clientes recorrentes representaram 5,82% da receita de itens com frete no período observado.

A receita média acumulada por cliente recorrente foi superior à dos clientes de compra única.

Esse resultado decorre principalmente da maior frequência de compra e não de ticket superior por pedido.

Importante:
Os resultados representam o comportamento observado dentro do período disponível e não devem ser interpretados como lifetime value definitivo do cliente.

[INSERIR SCREENSHOT 12 — comparação entre clientes recorrentes e compra única]


### 8.4 Desempenho logístico global

Foram observados:

- 99.441 pedidos;
- 96.476 pedidos entregues;
- 2.965 sem data de entrega;
- 6.535 pedidos atrasados;
- tempo médio de entrega de aproximadamente 12,5 dias;
- mediana de 10 dias;
- percentil 90 de 23 dias;
- taxa de atraso de aproximadamente 6,77% entre pedidos entregues.

A distribuição apresenta cauda longa, com alguns pedidos apresentando tempos de entrega muito superiores à mediana.

A ausência de data de entrega não foi automaticamente interpretada como falha logística, pois pode refletir diferentes estados do ciclo do pedido.

[INSERIR SCREENSHOT 13 — indicadores logísticos globais]


### 8.5 Logística por UF

A análise geográfica mostrou que incidência percentual e impacto absoluto precisam ser avaliados conjuntamente.

Exemplos:

- AL apresentou taxa de atraso elevada, porém menor número absoluto de pedidos atrasados;
- SP apresentou taxa proporcional menor, porém grande volume absoluto de atrasos devido ao alto número de pedidos;
- RJ combinou volume elevado com taxa de atraso superior à média global.

SP e RJ concentraram juntos aproximadamente metade dos pedidos atrasados observados.

Esses resultados não estabelecem causalidade geográfica. A UF deve ser interpretada como dimensão de segmentação operacional.

[INSERIR SCREENSHOT 14 — logística por UF]


### 8.6 Sellers

Como um pedido pode conter itens de múltiplos vendedores, a análise foi construída no grão seller × pedido.

Dessa forma, a presença de um seller em um pedido atrasado não significa que ele tenha causado o atraso.

A análise serve para identificar exposição ou associação com pedidos atrasados.

No RJ, os atrasos se mostraram relativamente pulverizados entre sellers, sem concentração extrema em poucos vendedores.

[INSERIR SCREENSHOT 15 — análise de sellers no RJ]


### 8.7 Categorias e atraso

Também foi construída uma análise no grão pedido × categoria.

Pedidos podem conter múltiplas categorias. Portanto, participações percentuais por categoria não devem ser somadas como se fossem partes exclusivas de um total.

Algumas categorias apresentaram maior taxa proporcional de atraso, enquanto categorias de grande volume apresentaram maior impacto absoluto.

Exemplos observados:

- audio apresentou taxa relevante de atraso, mas pequeno impacto absoluto;
- bed_bath_table apresentou grande quantidade absoluta de atrasos;
- health_beauty também apresentou impacto absoluto relevante;
- office_furniture apresentou ciclo de entrega mais longo.

[INSERIR SCREENSHOT 16 — logística por categoria]


### 8.8 Características físicas dos produtos

Foi avaliada a associação entre características físicas dos produtos e indicadores logísticos.

Para peso:

- Q1: 5,93% dos itens pertenciam a pedidos atrasados;
- Q4: 7,16%;
- tempo médio de entrega aumentou de 11,50 para 13,49 dias entre Q1 e Q4.

Para volume:

- não houve progressão consistente nos três primeiros quartis;
- o quarto quartil apresentou 7,04% de itens em pedidos atrasados;
- tempo médio de entrega no Q4 foi de 13,49 dias.

A análise indica associação descritiva, mas não permite afirmar causalidade.

[INSERIR SCREENSHOT 17 — quartis de peso e volume]


## 9. Discussão Geral

O pipeline permitiu responder às perguntas propostas combinando perspectivas comerciais, de clientes e logísticas.

O valor da camada Gold não está apenas na consolidação dos dados, mas na possibilidade de responder perguntas em granularidades distintas com maior controle sobre duplicidade e integridade.

Os principais aprendizados foram:

1. granularidade deve ser definida antes da análise;
2. fatos com grãos diferentes não devem ser combinados diretamente sem cuidado;
3. taxa percentual e impacto absoluto podem levar a conclusões operacionais diferentes;
4. recorrência deve ser analisada com customer_unique_id;
5. atraso deve ser analisado no grão de pedido;
6. seller e categoria não devem ser tratados como causa direta do atraso;
7. associações com peso e volume exigem cautela de interpretação.

O projeto demonstrou a construção de um pipeline funcional, reproduzível e orientado a perguntas de negócio.


## 10. Limitações

Principais limitações do MVP:

- utilização de base histórica pública e estática;
- ausência de atualização incremental;
- ausência de dados externos de transportadora, clima, malha logística ou distância real percorrida;
- pedidos com múltiplos sellers dificultam atribuição de responsabilidade por atraso;
- pedidos com múltiplas categorias geram sobreposição em análises por categoria;
- presença de registros sem categoria;
- presença de pedidos sem data real de entrega;
- análises físicas demonstram associação, não causalidade;
- métricas de recorrência representam apenas o período observado.


## 11. Autoavaliação

O objetivo principal do MVP foi atingido.

Foi possível construir um pipeline completo em ambiente Databricks, cobrindo ingestão, diagnóstico de qualidade, transformação, modelagem, agregações, validação e análise.

Entre as principais dificuldades encontradas estiveram:

- definição correta da granularidade das tabelas;
- tratamento de relações muitos-para-muitos;
- construção e validação das regras de atraso;
- inclusão correta de customer_id na dim_order;
- reconciliação entre pedidos, itens e pagamentos;
- interpretação de métricas com sobreposição de sellers e categorias;
- organização dos notebooks em responsabilidades independentes.

Ao longo do desenvolvimento, essas dificuldades levaram a melhorias na arquitetura.

A solução final separa:

- ingestão;
- qualidade;
- transformação;
- modelagem dimensional;
- agregações;
- Quality Gate;
- análise.

Essa organização tornou o pipeline mais reproduzível e tecnicamente mais defensável.

Também foi possível evoluir a análise para além de indicadores simples, discutindo granularidade, limitações e risco de interpretação causal indevida.

Como aprendizado principal, o projeto reforçou que engenharia de dados não consiste apenas em transformar dados, mas em garantir que as estruturas criadas permitam responder perguntas de forma confiável e rastreável.


## 12. Trabalhos Futuros

Possíveis evoluções:

- ingestão incremental;
- automação baseada em chegada de arquivos;
- controle de schema evolution;
- testes de qualidade com histórico de execução;
- dashboards em Power BI ou Databricks SQL;
- monitoramento operacional do pipeline;
- análise espacial mais detalhada;
- incorporação de distância logística;
- estudo de SLA por seller;
- investigação de causalidade com dados externos;
- criação de modelos de previsão de atraso;
- implementação de CI/CD para notebooks e jobs;
- versionamento formal de contratos de dados.


## 13. Estrutura do Repositório

Sugestão de organização:

/
├── notebooks/
│   ├── 01_ingestao_bronze.ipynb
│   ├── 02_qualidade_dados_bronze.ipynb
│   ├── 03_transformacao_silver.ipynb
│   ├── 04_modelagem_gold.ipynb
│   ├── 05_agregacoes_gold.ipynb
│   ├── 06_validacao_gold.ipynb
│   └── 07_analise_gold.ipynb
│
├── docs/
│   ├── Catalogo_Dados_MVP_Olist.xlsx
│   └── screenshots/
│
└── README.txt


## 14. Evidências Recomendadas

Para a entrega final, incluir screenshots das seguintes etapas:

1. fonte/base utilizada;
2. tabelas Bronze persistidas;
3. arquitetura do Workflow;
4. tabelas Gold no catálogo;
5. catálogo de dados;
6. Workflow completo;
7. execução com status Succeeded;
8. Quality Gate;
9. execução da validação Gold;
10. vendas mensais;
11. categorias;
12. recorrência;
13. logística global;
14. logística por UF;
15. sellers;
16. categorias e atraso;
17. peso e volume.

É preferível utilizar evidências legíveis e diretamente relacionadas aos critérios do trabalho, evitando screenshots redundantes.


## 15. Conclusão

O MVP atingiu o objetivo de construir um pipeline de dados funcional em nuvem, transformando dados brutos de e-commerce em estruturas confiáveis e prontas para análise.

A arquitetura Medallion permitiu separar ingestão, tratamento e consumo.

A camada Gold disponibilizou dimensões, fatos e agregados capazes de responder perguntas comerciais, de comportamento de clientes e de logística.

A execução do Workflow e do Quality Gate demonstrou que o pipeline final é reproduzível e consistente dentro das regras estabelecidas.

O resultado final representa não apenas uma análise da base Olist, mas uma implementação prática do ciclo de Engenharia de Dados:

fonte → ingestão → qualidade → transformação → modelagem → validação → análise.


## Disponibilidade dos Dados

Os dados utilizados neste MVP são públicos e foram obtidos da base Olist.

Por questões de tamanho, os arquivos CSV não são versionados diretamente
no repositório Git.

Para baixar os arquivos utilizados, o link a ser utlizado: https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce?resource=download

Os dados também podem ser obtidos diretamente na fonte pública original,
informada na seção de Coleta dos Dados.

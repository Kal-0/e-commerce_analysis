# Pipeline de Dados — Descrição Detalhada

## Metodologia

O pipeline segue as etapas obrigatórias de um pipeline de Big Data, implementadas sobre a plataforma **Databricks** com arquitetura **Medallion**. Não há execução local — todo o processamento é realizado no ambiente distribuído do Databricks.

---

## Etapas do Pipeline

### 1. Fonte de Dados (Data Source)

**Dataset:** [Instacart Market Basket Analysis](https://www.kaggle.com/datasets/psparks/instacart-market-basket-analysis) — Kaggle

Dados estruturados em formato CSV, descrevendo o histórico de compras de mais de 200 mil clientes em um supermercado online. Inclui informações de pedidos, produtos, corredores, departamentos e padrões de recompra.

Volumes dos dados:
- ~3,4 milhões de pedidos
- ~32 milhões de itens em pedidos
- ~50 mil produtos catalogados

---

### 2. Ingestão (Ingestion)

**Tipo:** Batch

**Notebook:** `SRC/ETL/0.RAW/ingest_Instacart_kaggle.ipynb`

O notebook realiza o download dos arquivos CSV do Kaggle via API e os armazena no volume do Databricks em `/Volumes/big_data/raw/data/instacart/`. Os arquivos são mantidos em formato original (CSV), sem modificações, como camada de auditoria imutável.

---

### 3. Armazenamento Bruto — BRONZE

**Notebook:** `SRC/ETL/1.BRONZE/update_bronze_tables.ipynb`

**Schema:** `big_data.bronze`

Os CSVs da camada RAW são lidos via Spark e gravados como Delta Tables no schema Bronze. Não são aplicadas transformações de negócio nesta etapa — o objetivo é garantir a persistência dos dados brutos em formato otimizado (Delta), com adição de metadados de controle (`ingestion_timestamp`, `source_file`).

---

### 4. Transformação (Transformation) — SILVER

**Notebook:** `SRC/ETL/2.SILVER/update_silver_tables.ipynb`

**Schema:** `big_data.silver`

Etapa central de qualidade e enriquecimento dos dados. As transformações incluem:

- **Type casting:** Conversão de tipos para os adequados (ex: IDs como inteiros, timestamps corretos).
- **Filtragem de nulos:** Remoção de registros inválidos.
- **Feature engineering:**
  - `is_first_order` — identifica o primeiro pedido de cada cliente.
  - `period_of_day` — classifica o horário do pedido em `morning`, `afternoon`, `evening` ou `night`.
  - `price_band` — categoriza produtos por faixa de preço em `Very Low`, `Low`, `Medium`, `High`, `Premium` ou `Luxury`.
- **Enriquecimento:** Join de `products` com `aisles`, `prices`, `departments` para gerar `products_enriched`.
- **Unificação:** Union de `order_products__prior` e `order_products__train` em uma única tabela `order_products`.

Metadado adicionado: `_silver_timestamp`.

---

### 5. Carregamento e Destino — GOLD

**Notebook:** `SRC/ETL/3.GOLD/analysis_gold_tables.ipynb`

**Schema:** `big_data.gold`

Camada analítica (em progresso). Os dados da Silver são agregados e transformados em tabelas de negócio para consumo por analistas e ferramentas de visualização.

Tabelas geradas:

**Sales Analytics**
- `sales_by_time_period` — total de pedidos, itens vendidos e percentual por período do dia.
- `sales_by_hour` — padrões de compra por hora, identificação de horários de pico.
- `reorder_analysis_by_department` — total de itens, itens recomprados e taxa de recompra por departamento.

**Customer Analytics**
- `customer_segmentation` — segmentos: New / Occasional / Regular / Frequent / Loyal.
- `customer_purchase_frequency` — distribuição por intervalo de dias entre compras (1-7, 8-14, 15-21, 22-30, 30+).

**Product Analytics**
- `product_performance` — vezes comprado, pedidos únicos, vezes recomprado, taxa de recompra.
- `department_performance` — produtos únicos, total vendido, pedidos únicos, itens recomprados, taxa de recompra, média de itens por pedido.
- `reorder_analysis_by_aisle` — total de itens, itens recomprados e taxa de recompra por corredor.

Os resultados ficam disponíveis como tabelas Delta no schema `big_data.gold`, consumíveis diretamente via SQL no Databricks ou por notebooks de visualização.

---

## Checklist

| Etapa | Status |
|---|---|
| Ingestão (RAW) | Finalizado |
| Armazenamento Bronze | Finalizado |
| Transformação Silver | Finalizado |
| Camada Gold (Analítica) | Finalizado |
| Machine Learning (Recomendação) | Finalizado |
| Dashboard & Visualização | Finalizado |

# ETL Workflows - Databricks Asset Bundles (DABs)

Este diretório contém configurações YAML para Databricks Workflows que orquestram o pipeline ETL completo usando Databricks Asset Bundles.

## 📁 Arquivos

| Arquivo | Propósito | Tasks | Path |
|---------|-----------|-------|------|
| `etl_bronze.yml` | Camada Bronze (ingestão raw) | 1 | `/Workspace/Shared/e-commerce_analysis/` |
| `etl_silver.yml` | Camada Silver (transformações) | 3 | `/Workspace/Shared/e-commerce_analysis/` |
| `etl_gold.yml` | Camada Gold (analytics) | 13 | `/Workspace/Shared/e-commerce_analysis/` |
| `etl_instacart_daily.yml` | **Orquestrador** (roda os 3 acima) | 3 | - |

## 🏗️ Arquitetura Modular

Cada camada tem seu próprio workflow independente:

```
etl_bronze.yml (1 task)
    ↓
etl_silver.yml (3 tasks)
    ↓
etl_gold.yml (13 tasks em paralelo)
```

O orquestrador combina tudo:

```
etl_instacart_daily.yml
    ├─→ run_bronze_layer → chama job bronze_layer
    ├─→ run_silver_layer → chama job silver_layer (após Bronze)
    └─→ run_gold_layer   → chama job gold_layer (após Silver)
```

## 📊 Estrutura do YAML (DABs)

Todos os arquivos seguem o formato Databricks Asset Bundles:

```yaml
resources:
  jobs:
    [job_name]:
      name: [job_name]
      tasks:
        - task_key: [task_name]
          notebook_task:
            notebook_path: /Workspace/Shared/e-commerce_analysis/...
            source: WORKSPACE
          min_retry_interval_millis: 900000
          disable_auto_optimization: true
      queue:
        enabled: true
      performance_target: PERFORMANCE_OPTIMIZED
```

### Características

* **Sem cluster explícito** - Usa compute padrão do workspace
* **Performance otimizado** - `performance_target: PERFORMANCE_OPTIMIZED`
* **Queue habilitado** - `queue.enabled: true`
* **Retry configurado** - `min_retry_interval_millis: 900000` (15 min)
* **Auto-optimization desabilitado** - `disable_auto_optimization: true`

## 🚀 Como Usar (Databricks Bundle)

### Passo 1: Fazer Deploy dos Jobs das Camadas

```bash
# Navegue até a pasta do projeto
cd /Workspace/Shared/e-commerce_analysis/SRC/ETL/WORKFLOWS

# Deploy Bronze
databricks bundle deploy --target bronze
# Output: Job ID: 123456

# Deploy Silver
databricks bundle deploy --target silver
# Output: Job ID: 123457

# Deploy Gold
databricks bundle deploy --target gold
# Output: Job ID: 123458
```

**Importante:** Anote cada job ID!

### Passo 2: Atualizar Orquestrador

Edite `etl_instacart_daily.yml` e substitua os placeholders:

```yaml
# ANTES:
run_job_task:
  job_id: ${bronze_job_id}

# DEPOIS:
run_job_task:
  job_id: 123456
```

Substitua os 3 placeholders:
* `${bronze_job_id}` → ID do job Bronze
* `${silver_job_id}` → ID do job Silver
* `${gold_job_id}` → ID do job Gold

### Passo 3: Deploy do Orquestrador

```bash
databricks bundle deploy --target orchestrator
```

## ▶️ Como Executar

### Opção 1: Executar Camadas Individuais

```bash
# Rodar só Bronze
databricks jobs run-now --job-id 123456

# Rodar só Silver (assume que Bronze já rodou)
databricks jobs run-now --job-id 123457

# Rodar só Gold (assume que Bronze + Silver já rodaram)
databricks jobs run-now --job-id 123458
```

### Opção 2: Executar Pipeline Completo (Orquestrador)

```bash
# Rodar tudo (Bronze → Silver → Gold)
databricks jobs run-now --job-id [orchestrator_job_id]
```

### Opção 3: Via Interface Databricks

1. Navegue para **Workflows** no Databricks
2. Encontre o job desejado
3. Clique em **Run Now**
4. Monitore na aba **Runs**

## 📋 Detalhes dos Workflows

### 1. Bronze Layer (etl_bronze.yml)

**Tabelas Criadas:**
* bronze.aisles
* bronze.departments
* bronze.order_products_prior
* bronze.order_products_train
* bronze.orders
* bronze.products

**Runtime:** ~10-20 minutos

### 2. Silver Layer (etl_silver.yml)

**Tasks:**
* `silver_orders` - 3.4M linhas
* `silver_products_enriched` - 50K linhas
* `silver_order_products` - 33M linhas (depende de silver_orders)

**Runtime:** ~20-30 minutos

### 3. Gold Layer (etl_gold.yml)

**Tasks (13 total - todas em paralelo):**

**Fact Tables:**
* gold_executive_summary
* gold_customer_segmentation
* gold_product_performance
* gold_product_pairs

**Views:**
* gold_sales_by_time_period
* gold_sales_by_hour
* gold_department_performance
* gold_reorder_analysis_by_department
* gold_customer_lifecycle_metrics
* gold_cross_category_patterns
* gold_customer_purchase_frequency
* gold_price_band_performance
* gold_add_to_cart_analysis

**Runtime:** ~15-30 minutos

### 4. Orquestrador (etl_instacart_daily.yml)

**Tasks:**
* run_bronze_layer
* run_silver_layer (depende Bronze)
* run_gold_layer (depende Silver)

**Runtime Total:** ~45-80 minutos

## 🔍 Monitoramento

### Via CLI:

```bash
# Listar todos os jobs
databricks jobs list

# Ver detalhes de um job
databricks jobs get --job-id [job_id]

# Listar execuções recentes
databricks jobs list-runs --job-id [job_id] --limit 10

# Ver detalhes de execução (com logs)
databricks jobs get-run --run-id [run_id]
```

### Via Interface:

1. **Workflows** → Selecionar job → **Runs**
2. Clicar na execução
3. Clicar na task para ver logs

## 🎯 Padrões de Uso

### Desenvolvimento:

1. Fazer mudanças em notebook
2. Testar localmente
3. Fazer commit
4. Re-deploy do bundle: `databricks bundle deploy --target [layer]`
5. Executar job para validar

### Produção:

* **Manual:** Executar orquestrador via UI ou CLI
* **Scheduled:** Adicionar schedule ao YAML e re-deploy

### Debug:

1. Identificar camada que falhou
2. Rodar camada independente
3. Corrigir notebook
4. Re-deploy
5. Re-executar

## 🔧 Customização

### Adicionar Schedule

Adicione ao YAML (dentro do job):

```yaml
resources:
  jobs:
    bronze_layer:
      name: bronze_layer
      schedule:
        quartz_cron_expression: "0 0 2 * * ?"
        timezone_id: "America/Sao_Paulo"
        pause_status: UNPAUSED
      tasks:
        ...
```

### Ajustar Retry Interval

```yaml
- task_key: my_task
  min_retry_interval_millis: 1800000  # 30 minutos
```

### Adicionar Max Retries

```yaml
- task_key: my_task
  max_retries: 3
  min_retry_interval_millis: 900000
```

## ✅ Checklist de Validação

Antes de produção:

* [ ] 4 arquivos YAML na pasta WORKFLOWS
* [ ] Jobs Bronze, Silver, Gold deployed com sucesso
* [ ] Job IDs anotados e atualizados no orquestrador
* [ ] Orquestrador deployed com sucesso
* [ ] Teste de execução individual das camadas OK
* [ ] Teste de execução do orquestrador OK
* [ ] Caminhos dos notebooks validados (`/Workspace/Shared/...`)

## 📈 Performance

### Volumes:

* Bronze: 6 tabelas, ~37M linhas
* Silver: 3 tabelas, ~36M linhas
* Gold: 13 tabelas/views

### Runtimes:

| Camada | Runtime | Gargalo |
|--------|---------|---------|
| Bronze | 10-20 min | Parse CSV |
| Silver | 20-30 min | order_products (33M) |
| Gold | 15-30 min | product_pairs (self-join) |
| **Total** | **45-80 min** | Execução sequencial |

## 🆘 Troubleshooting

### Erro: "Notebook not found"
**Causa:** Caminho incorreto  
**Fix:** Verificar que path começa com `/Workspace/Shared/...`

### Erro: "Job not found" (orquestrador)
**Causa:** Job ID incorreto  
**Fix:** `databricks jobs list`, copiar ID correto, re-deploy

### Erro: "Task timeout"
**Causa:** Task excedeu limite padrão  
**Fix:** Adicionar `timeout_seconds` na task

## 📚 Recursos

* [Databricks Asset Bundles Docs](https://docs.databricks.com/dev-tools/bundles/index.html)
* [Databricks Jobs API](https://docs.databricks.com/dev-tools/api/latest/jobs.html)
* [Databricks CLI](https://docs.databricks.com/dev-tools/cli/index.html)

## 📞 Suporte

Para issues ou dúvidas:
* Checar logs dos jobs na UI
* Revisar AGENTS.md para padrões de notebooks
* Contato: ccbh@cesar.school

---

**Versão:** 3.0 (DABs)  
**Última Atualização:** 2026-05-31  
**Formato:** Databricks Asset Bundles  
**Localização:** `/Workspace/Shared/e-commerce_analysis/SRC/ETL/WORKFLOWS/`

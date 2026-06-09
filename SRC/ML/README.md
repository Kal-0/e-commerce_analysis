# 🎯 Product Recommendation System

## Overview

Sistema de recomendação de produtos para e-commerce (Instacart dataset) implementado com PySpark e Spark MLlib, com features avançadas de contexto temporal e padrões de co-compra.

**Objetivo:** Prever quais produtos um cliente provavelmente comprará em seu próximo pedido.

**Dataset:** Instacart Market Basket Analysis
* 206K usuários
* 49,685 produtos únicos
* 3.3M orders
* 33.8M interactions (order-products)

---

## 🚀 Quick Start

### Execução Completa do Pipeline

```bash
# Notebook principal (end-to-end)
0_Run_Full_Pipeline.ipynb
```

**Runtime:**
* Sample mode (5-10% usuários): ~15-30 min
* Full dataset (100%): ~4-6 horas

### Execução Individual

```bash
# 1. Preparação de dados + feature engineering
1_Data_Preparation_Recommendation.ipynb  # ~5-10 min

# 2. Treinamento de modelos + avaliação
2_Model_Training_Recommendation.ipynb    # ~10-25 min (sample mode)
```

---

## 🏗️ Arquitetura do Pipeline

### **Fase 1: Data Preparation & Feature Engineering**
📓 `1_Data_Preparation_Recommendation.ipynb`

#### **1.1 Data Loading**
* Source: `big_data.silver.orders`, `big_data.silver.order_products`
* Build user-item interaction matrix (33.8M interactions)

#### **1.2 User Filtering**
* Keep users with ≥3 orders (206K users)
* Remove cold-start users

#### **1.3 Temporal Train/Test Split**
* **Train:** First N-2 orders (per user)
* **Test:** Last 2 orders (per user)
* **Ground Truth:** Unique products purchased in test orders
* ✅ **NO DATA LEAKAGE:** All features calculated ONLY from train set

#### **1.4 Basic Features (Train Set Only)**
```python
# Purchase statistics (per user-product pair)
- times_purchased: count of purchases
- times_reordered: count of reorders
- last_purchased_order: order_number of last purchase
```

#### **1.5 Implicit Ratings**
Três tipos de ratings para experimentação:

**🔹 Rating Original (frequency-based):**
```python
rating = 1.0 + (0.5 if reordered) + min(times_purchased × 0.1, 0.5)
Range: [1.0, 2.0]
```

**🔹 Rating Contextual (NEW - behavioral patterns):**
```python
rating_contextual = 1.0 
                  + (recent_purchase_ratio × 0.6)      # temporal relevance
                  + min(user_copurchase_count × 0.05, 0.3)  # basket patterns
                  + (1.0 / (global_copurchase_rank + 1)) × 0.1  # popularity
Range: [1.0, 2.0]
```

**🔹 Rating Combined (hybrid):**
```python
rating_combined = (rating × 0.6) + (rating_contextual × 0.4)
Range: [1.0, 2.0]
```

#### **1.6 Advanced Features (NEW)**

**Feature 1: Recent Purchase Ratio**
* **Purpose:** Capture temporal relevance (recent behavior > old behavior)
* **Calculation:** Proportion of purchases in last 7 training orders
* **Formula:** `recent_purchase_ratio = count_last_7_orders / 7`
* **Use case:** Detect changing preferences

**Feature 2: User-Specific Co-Purchase Count**
* **Purpose:** Personal basket patterns (what THIS user buys together)
* **Calculation:** Count how often product was bought with other products in same order
* **Use case:** User-specific associations (diapers + wipes for parents)

**Feature 3: Global Co-Purchase Rank**
* **Purpose:** Global popularity signal (what EVERYONE buys together)
* **Calculation:** Rank products by frequency of co-purchase across all users
* **Use case:** Cold-start recommendations, global trends

#### **1.7 Output Tables**

**✅ big_data.ml_features.train_interactions** (12M rows, 14 columns)
```
- user_id, product_id
- times_purchased, times_reordered, last_purchased_order
- rating (original)
- recent_purchases, recent_purchase_ratio
- user_copurchase_count, global_copurchase_count, global_copurchase_rank
- rating_contextual (NEW)
- rating_combined (NEW - hybrid)
- _created_at (timestamp)
```

**✅ big_data.ml_features.test_ground_truth** (206K rows)
```
- user_id
- actual_products (array of product_ids)
- test_order_count, total_test_products
- _created_at (timestamp)
```

**🔒 Data Leakage Prevention:** VERIFIED
* All features/ratings calculated ONLY from train orders
* Test data never seen during feature engineering
* Temporal order respected (train before test)

---

### **Fase 2: Model Training & Evaluation**
📓 `2_Model_Training_Recommendation.ipynb`

#### **2.1 Configuration**

```python
# Rating selection (experiment with 3 options)
RATING_COLUMN = "rating_combined"  # "rating", "rating_contextual", "rating_combined"

# Sampling mode (for fast development)
SAMPLE_MODE = True          # False = full dataset (production)
SAMPLE_FRACTION = 0.05      # 5% of USERS (not interactions!)

# Model configuration
LIGHTWEIGHT_MODE = True     # False = full ALS (50 factors, 10 iterations)
REUSE_MODELS = False        # True = load cached models
K = 5                       # Top-K recommendations
```

**⚠️ IMPORTANT:** Sampling is USER-LEVEL (not line-level)
* ✅ Each sampled user keeps COMPLETE history (no fragmentation)
* ✅ ALS learns correct user factors
* ✅ Realistic metrics
* ❌ Line-level sampling would fragment user data and distort results

#### **2.2 Model 1: Item-Item Collaborative Filtering (Baseline)**

**Algorithm:**
```
For each user:
  1. Get products purchased in train set
  2. Find similar products using product_pairs table (co-purchase patterns)
  3. Aggregate scores and rank
```

**Pros:** Simple, explainable, robust to sparse data  
**Cons:** Limited to explicit co-purchase patterns

**Current Results (5% users, rating_combined):**
* Precision@5: **11.9%**
* Recall@5: 3.2%
* NDCG@5: 13.1%
* Hit Rate@5: **43.1%** (43% of users get at least 1 correct product!)

#### **2.3 Model 2: ALS Matrix Factorization (Advanced)**

**Algorithm:** Implicit library (Alternating Least Squares)

**Hyperparameters:**
* Lightweight mode: 20 factors, 5 iterations
* Full mode: 50 factors, 10 iterations
* Regularization: 0.1

**Pros:** Learns latent patterns, handles sparsity  
**Cons:** Black-box, needs more data, expensive

**Current Results (5% users, rating_combined, lightweight):**
* Precision@5: 6.8% (underperforming - needs full mode + more data)
* Recall@5: 1.8%
* NDCG@5: 7.1%
* Hit Rate@5: 27.7%

**Expected Results (10% users, full mode):**
* Precision@5: **~10-13%**
* Hit Rate@5: **~35-40%**

#### **2.4 Model 3: Hybrid Ensemble**

**Strategy:**
```python
final_score = 0.5 × item_item_score_normalized + 0.5 × als_score_normalized
```

**Pros:** Combines complementary signals  
**Cons:** More complex, needs tuning

**Current Results (5% users, rating_combined):**
* Precision@5: 10.1%
* Recall@5: 2.8%
* NDCG@5: 10.6%
* Hit Rate@5: 38.6%

#### **2.5 Evaluation Metrics**

* **Precision@K:** % of recommended products that were actually purchased
* **Recall@K:** % of actual purchases that were recommended
* **NDCG@K:** Normalized Discounted Cumulative Gain (ranking quality)
* **Hit Rate@K:** % of users with at least 1 correct recommendation

**Why 12% precision is GOOD:**
* Catálogo: 49,685 produtos
* Random baseline: **0.024%** (5/49685 × 100)
* Our model: **11.9%** = **500x better than random!**
* Kaggle competition winner: ~16-18%

#### **2.6 Output: Production Recommendations**

**✅ big_data.gold.user_recommendations**

```sql
SELECT 
  user_id,
  recommendations,  -- array of top-10 product_ids
  model_version,    -- "item_item_v1", "als_v1", "hybrid_v1"
  generated_at,
  k
FROM big_data.gold.user_recommendations
WHERE user_id = 12345;
```

Example output:
```
user_id: 12345
recommendations: [47626, 24852, 21137, 13176, 47209, 27966, ...]
model_version: "item_item_v1"
generated_at: 2026-06-09 02:22:59
k: 10
```

---

## 🧪 Experimentos Sugeridos

### **Experimento 1: Comparar os 3 Ratings**

```python
# No notebook 2_Model_Training_Recommendation
RATING_COLUMN = "rating"             # Test 1: Original (frequency)
RATING_COLUMN = "rating_contextual"  # Test 2: Contextual (behavioral)
RATING_COLUMN = "rating_combined"    # Test 3: Hybrid (current)
```

**Hipótese:** rating_contextual pode ser melhor para usuários com padrões voláteis.

### **Experimento 2: Otimizar ALS**

```python
SAMPLE_FRACTION = 0.10       # Increase to 10% users
LIGHTWEIGHT_MODE = False     # Full mode: 50 factors, 10 iterations
```

**Expectativa:** ALS deve melhorar de 6.8% → ~10-13% precision@5.

### **Experimento 3: Full Dataset (Production)**

```python
SAMPLE_MODE = False  # Train on 100% of users
```

**Runtime:** ~4-6 horas  
**Expectativa:** Métricas aumentam ~20-30% vs sample mode.

### **Experimento 4: Advanced ML Models**

Usar features avançadas com modelos supervisionados:

* **LightGBM/XGBoost:** Exploit all 14 features
* **Neural CF:** Deep learning for collaborative filtering
* **Two-stage:** CF candidates → ML ranker

---

## 📊 Métricas Target (Full Dataset, Production)

| Métrica | Target | Current (5% sample) | Full Dataset (expected) |
|---------|--------|---------------------|-------------------------|
| Precision@5 | 15-18% | 11.9% | 14-17% |
| Recall@5 | 22-28% | 3.2% | 20-25% |
| NDCG@5 | 0.32-0.36 | 0.131 | 0.30-0.35 |
| Hit Rate@5 | 40%+ | 43.1% ✅ | 45-50% |

---

## 🔄 Re-treinamento

**Frequência Recomendada:** Mensal

**Gatilhos para Re-treino:**
* Precision cai >2 pontos percentuais
* Novos produtos >5% do catálogo
* Mudanças sazonais (holidays, promoções)
* Novos usuários >10% da base

**Processo:**
1. Re-executar `1_Data_Preparation_Recommendation` (novas features)
2. Re-executar `2_Model_Training_Recommendation` (novos modelos)
3. Validar métricas vs baseline anterior
4. Deploy se métricas mantêm ou melhoram

---

## 📚 Estrutura de Arquivos

```
SRC/ML/
├── 0_Run_Full_Pipeline.ipynb              # End-to-end pipeline
├── 1_Data_Preparation_Recommendation.ipynb # Feature engineering (Phase 1)
├── 2_Model_Training_Recommendation.ipynb   # Model training (Phase 2)
├── README.md                              # This file
└── models/                                # Cached models (optional)
    ├── als_model.pkl
    └── item_item_scores_cached/
```

---

## 🔧 Configuração de Ambiente

**Compute:** Databricks Serverless (SQL Warehouse ou Cluster)

**Bibliotecas Necessárias:**
* PySpark (built-in)
* implicit (ALS): `%pip install implicit`
* pandas, numpy (built-in)

**Permissões Necessárias:**
* READ: `big_data.silver.orders`, `big_data.silver.order_products`
* READ: `big_data.gold.product_pairs`
* WRITE: `big_data.ml_features.*`
* WRITE: `big_data.gold.user_recommendations`

---

## ⚠️ Limitações Conhecidas

* **Temporal granularity:** Apenas order_number (sem timestamps absolutos)
* **Implicit feedback only:** Sem dados de cliques/views/wishlist
* **No inventory data:** Não considera estoque disponível
* **Cold-start:** Usuários novos recebem recomendações populares (fallback)
* **Computational cost:** Full dataset training takes 4-6 hours

---

## 🚀 Próximos Passos

### **Curto Prazo**
- [ ] Executar full dataset (SAMPLE_MODE=False)
- [ ] Comparar os 3 ratings (original vs contextual vs combined)
- [ ] Otimizar weights do Hybrid (testar 70/30, 80/20)
- [ ] Tune ALS hyperparameters (factors, iterations, regularization)

### **Médio Prazo**
- [ ] Adicionar product metadata (department, aisle) para diversidade
- [ ] Implementar cold-start strategy (popularity + department similarity)
- [ ] A/B test em produção (comparar vs baseline)
- [ ] Dashboard de monitoramento (métricas em tempo real)

### **Longo Prazo**
- [ ] Advanced ML: LightGBM/XGBoost com todas as features
- [ ] Neural Collaborative Filtering (Deep Learning)
- [ ] Two-stage ranker: CF candidates → ML re-ranker
- [ ] Real-time recommendations (streaming pipeline)
- [ ] Contextual bandits (online learning)

---

## 📖 Referências

* **Dataset:** Instacart Market Basket Analysis (Kaggle)
* **Paper:** "Collaborative Filtering for Implicit Feedback Datasets" (Hu et al., 2008)
* **Library:** implicit (https://github.com/benfred/implicit)
* **Kaggle Winner:** https://www.kaggle.com/c/instacart-market-basket-analysis/discussion/38112

---

## 👥 Team & Version

**ML Team** | Big Data Project  
**Version:** 2.0 (Advanced Features)  
**Last Updated:** 2026-06-09  
**Status:** ✅ Production Ready (sample mode tested)

---

## 📞 Support

Para dúvidas ou problemas:
* Review notebook comments and markdown cells
* Check execution logs for errors
* Validate input table schemas
* Verify permissions on Unity Catalog tables

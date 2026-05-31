# Gold Layer - View Template Guide

## Overview

This template provides a **simple, standardized structure** for creating SQL views in the Gold layer. Unlike fact tables that require extensive Python transformations and validations, **views are lightweight definitions** that query underlying tables in real-time.

**Template Location:** `/Users/ccbh@cesar.school/e-commerce_analysis/SRC/ETL/TEMPLATES/view_template`

---

## 🎯 When to Use Views vs Tables

### Use a **VIEW** when:
* ✅ Result set is **small** (< 1M rows)
* ✅ Query is **fast** (< 30 seconds)
* ✅ Data must be **always current** (real-time)
* ✅ Logic is **simple** (basic joins, aggregations)
* ✅ **No complex transformations** needed
* ✅ Storage overhead is a concern

### Use a **TABLE** when:
* ❌ Result set is **large** (> 1M rows)
* ❌ Query is **slow** (> 30 seconds)
* ❌ Scheduled refresh is acceptable
* ❌ **Complex transformations** with multiple steps
* ❌ Needs **data quality validations**
* ❌ Performance optimization is critical

---

## 📋 View Notebook Structure (3 Cells)

All view notebooks follow a simple **3-cell structure**:

### Cell 1: README (Markdown)
**Purpose:** Complete documentation of the view

**Required Sections:**
- **Purpose:** Business value and analytical questions answered
- **Type:** Always "SQL View (lightweight, real-time query)"
- **Input:** Source tables with row counts
- **Output:** View name, expected row count, refresh frequency
- **Use Cases:** 3-5 bullet points of how the view is used
- **Why View (not Table)?:** Justification for using a view
- **SQL Logic:** Step-by-step description of the query
- **Execution:** Expected runtime

---

### Cell 2: Create View (SQL)
**Purpose:** Define the view using CREATE OR REPLACE VIEW

**Pattern:**
```sql
%sql
-- [View Name] View
-- Purpose: [One-line description]

CREATE OR REPLACE VIEW [catalog].[schema].vw_[view_name] AS
SELECT 
  [columns],
  [aggregations]
FROM [source_tables]
WHERE [filters]
GROUP BY [grouping]
ORDER BY [sorting];
```

**Key Points:**
- Always use `CREATE OR REPLACE VIEW` (idempotent)
- View name follows pattern: `vw_[descriptive_name]`
- Use clear aliases for tables
- Add comments for complex logic
- Use ROUND() for decimal precision
- Always include ORDER BY for consistent results

---

### Cell 3: Verify and Preview (SQL)
**Purpose:** Test that the view was created and preview results

**Pattern:**
```sql
%sql
-- Verify view exists and preview results
SELECT * FROM [catalog].[schema].vw_[view_name]
LIMIT 10;
```

**Optional Variations:**
- Add WHERE clause to filter specific records
- Use ORDER BY to show most relevant results first
- Remove LIMIT if result set is always small

---

## 🚀 Using the Template

### Quick Start (5 Steps)

1. **Copy the template notebook** to your target Gold layer directory
   - Location: `/Users/ccbh@cesar.school/e-commerce_analysis/SRC/ETL/3.GOLD/`
   - Naming: `gold_vw_[descriptive_name]`

2. **Update Cell 1 (README)**
   - Replace `[View Name]` with actual name
   - Fill in Purpose, Input, Output sections
   - List 3-5 Use Cases
   - Justify why a view (not table)
   - Describe SQL Logic steps

3. **Update Cell 2 (Create View)**
   - Replace placeholders: `[catalog].[schema].vw_[view_name]`
   - Write your SELECT statement
   - Add JOINs, WHERE, GROUP BY, ORDER BY as needed
   - Use ROUND() for metrics

4. **Update Cell 3 (Verify)**
   - Update view name in SELECT statement
   - Adjust LIMIT or add filters as appropriate

5. **Test Execution**
   - Run all 3 cells sequentially
   - Verify view is created in catalog
   - Check that results match expectations

---

## 📏 Naming Conventions

### Notebook Names
**Pattern:** `gold_vw_[descriptive_name]`

**Examples:**
- `gold_vw_sales_by_hour`
- `gold_vw_customer_lifecycle_metrics`
- `gold_vw_department_performance`

### View Names (in Catalog)
**Pattern:** `vw_[descriptive_name]` (without schema prefix)

**Examples:**
- `vw_sales_by_hour`
- `vw_customer_lifecycle_metrics`
- `vw_department_performance`

**Note:** View names do NOT include the schema name (`gold_`). The schema is already part of the fully qualified name: `big_data.gold.vw_sales_by_hour`

---

## 🎨 SQL Style Guide

### General Rules
- Use **uppercase** for SQL keywords (SELECT, FROM, WHERE, etc.)
- Use **lowercase** for table/column names
- Indent nested queries and clauses
- One column per line in SELECT
- Add blank lines between major clauses

### Example - Well Formatted:
```sql
CREATE OR REPLACE VIEW big_data.gold.vw_sales_by_hour AS
SELECT 
  o.order_hour_of_day,
  COUNT(DISTINCT o.order_id) AS total_orders,
  COUNT(op.product_id) AS total_items,
  ROUND(SUM(p.price_usd), 2) AS estimated_revenue_usd,
  ROUND(AVG(p.price_usd), 2) AS avg_item_price_usd
FROM big_data.silver.orders o
JOIN big_data.silver.order_products op ON o.order_id = op.order_id
JOIN big_data.silver.products_enriched p ON op.product_id = p.product_id
GROUP BY o.order_hour_of_day
ORDER BY o.order_hour_of_day;
```

### Common Patterns

#### Aggregation View
```sql
SELECT 
  dimension_column,
  COUNT(*) AS total_count,
  ROUND(AVG(metric), 2) AS avg_metric,
  ROUND(SUM(metric), 2) AS sum_metric
FROM source_table
GROUP BY dimension_column
ORDER BY total_count DESC;
```

#### Join View
```sql
SELECT 
  t1.id,
  t1.name,
  t2.attribute,
  t3.metric
FROM table1 t1
LEFT JOIN table2 t2 ON t1.id = t2.foreign_id
LEFT JOIN table3 t3 ON t1.id = t3.foreign_id
WHERE t1.status = 'active'
ORDER BY t1.name;
```

#### CTE View (for complex logic)
```sql
CREATE OR REPLACE VIEW big_data.gold.vw_complex_analysis AS
WITH step1 AS (
  SELECT 
    id,
    SUM(amount) AS total
  FROM table1
  GROUP BY id
),
step2 AS (
  SELECT 
    s1.id,
    s1.total,
    t2.category
  FROM step1 s1
  JOIN table2 t2 ON s1.id = t2.id
)
SELECT 
  category,
  COUNT(*) AS count,
  ROUND(AVG(total), 2) AS avg_total
FROM step2
GROUP BY category
ORDER BY avg_total DESC;
```

---

## ❌ Common Mistakes to Avoid

### 1. Using Fact Table Structure for Views
**Wrong:** 10 cells with Python transformations, validations, etc.
**Right:** 3 cells - README, CREATE VIEW, Verify

### 2. Including Schema in View Name
**Wrong:** `CREATE VIEW big_data.gold.gold_vw_sales`
**Right:** `CREATE VIEW big_data.gold.vw_sales`

### 3. No ORDER BY Clause
**Wrong:** Results in random order
**Right:** Always include ORDER BY for deterministic results

### 4. Not Using ROUND() for Decimals
**Wrong:** `AVG(price)` → Returns many decimal places
**Right:** `ROUND(AVG(price), 2)` → Returns 2 decimal places

### 5. Missing Documentation
**Wrong:** Empty or minimal README
**Right:** Complete documentation with Purpose, Use Cases, SQL Logic

---

## ✅ Quality Checklist

Before considering a view notebook complete, verify:

- [ ] **Notebook name** follows pattern: `gold_vw_[name]`
- [ ] **View name** follows pattern: `vw_[name]` (no schema prefix)
- [ ] **Cell 1 (README)** has all required sections
- [ ] **Cell 2 (CREATE VIEW)** uses CREATE OR REPLACE
- [ ] **Cell 2** includes ORDER BY clause
- [ ] **Cell 2** uses ROUND() for decimal metrics
- [ ] **Cell 3 (Verify)** has appropriate LIMIT or filter
- [ ] **All placeholders** replaced with actual values
- [ ] **SQL formatting** follows style guide
- [ ] **Execution tested** - all cells run successfully
- [ ] **Results verified** - output matches expectations

---

## 📊 View vs Table Comparison

| Aspect | View Notebook | Table Notebook |
|--------|---------------|----------------|
| **Cells** | 3 | 10 |
| **Language** | SQL-only | Python + SQL |
| **Validations** | None | Technical + Business |
| **Transformations** | Simple SQL | Complex Python |
| **Storage** | Zero (virtual) | Full Delta table |
| **Refresh** | Real-time | On-demand/scheduled |
| **Performance** | Fast (small data) | Fast (large data, materialized) |
| **Use Case** | Quick aggregations | Heavy transformations |

---

## 📞 Support

### Questions?
- Review existing view notebooks in `/Users/ccbh@cesar.school/e-commerce_analysis/SRC/ETL/3.GOLD/`
- Check `AGENTS.md` for general ETL standards
- Consult with Data Engineering team

### Improvements?
- Update this guide (VIEW_TEMPLATE_GUIDE.md)
- Update the template notebook (view_template)
- Share with the team

---

**Version:** 1.0  
**Created:** 2026-05-31  
**Author:** Data Engineering Team  
**Template:** `/Users/ccbh@cesar.school/e-commerce_analysis/SRC/ETL/TEMPLATES/view_template`

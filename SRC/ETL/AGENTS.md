# ETL Pipeline - Databricks Notebook Standards

## Overview

This document defines the **modular, robust, and maintainable** standards for all ETL notebooks in the Silver layer (and adaptable to Bronze/Gold). These patterns ensure consistency, data quality, and scalability across the entire data pipeline.

**Template Locations:**
* **Fact/Dimension Tables:** `SRC/ETL/TEMPLATES/table_templates`
* **Views:** `SRC/ETL/TEMPLATES/view_template`

---

## 🎯 Design Principles

### 1. **Modularity**
- One notebook per table or view
- Clear separation between transformation logic and validation logic
- Reusable UTILS module for common functions

### 2. **Data Quality First**
- Validations separated into Technical (schema integrity) and Business (domain rules)
- Conditional persistence: write to Delta **only if all validations pass**
- Comprehensive checks: NOT NULL, UNIQUE, RANGE, EXPECTED VALUES, REFERENTIAL INTEGRITY

### 3. **Configuration-Driven**
- All table names, schemas, and validation columns defined in a single Configuration cell
- Easy to adapt template for new tables
- Variables used throughout (avoid hardcoding)

### 4. **Serverless Compatibility**
- All code compatible with Databricks Serverless compute
- No RDD operations (use DataFrame API)
- No cache/persist operations (not supported on Serverless)
- Efficient use of Spark DataFrame API

---

## 📚 Notebook Types

### Fact/Dimension Tables
For **materialized tables** that require complex transformations, data quality validations, and conditional persistence.

**Use when:**
* Result set is large (> 1M rows)
* Complex transformations needed
* Data quality validations required
* Query performance is critical

**Template:** `SRC/ETL/TEMPLATES/table_templates`

**Key Sections:**
* README with complete documentation
* Setup: Dependencies, Imports, Configuration
* Transformation: Multi-step data processing
* Data Quality: Technical and Business validations
* Conditional Persistence to Delta

---

### Views
For **lightweight SQL views** that query underlying tables in real-time without materialization.

**Use when:**
* Result set is small (< 1M rows)
* Query is fast (< 30 seconds)
* Data must be always current (real-time)
* Simple SQL logic (basic joins, aggregations)
* No complex transformations needed

**Template:** `SRC/ETL/TEMPLATES/view_template`

**Guide:** `SRC/ETL/TEMPLATES/VIEW_TEMPLATE_GUIDE.md`

---

## 📊 Validation Separation: Technical vs Business

### Why Separate?

| Aspect | Technical DQ | Business DQ |
|--------|--------------|-------------|
| **Focus** | Schema integrity | Domain rules |
| **Owner** | Data Engineers | Product/Business |
| **Examples** | PK NULL, duplicates | Invalid status, price < 0 |
| **Failure** | ETL breaks | Wrong insights |
| **Fix** | Code/pipeline | Source data/rules |

### Benefits:

1. **Clear Ownership** - Engineers vs Business teams
2. **Faster Debugging** - Know where to look
3. **Better Reporting** - Separate quality metrics
4. **Modularity** - Can skip business checks in dev

---

## 🗂️ UTILS Module

### Location
`SRC/ETL/UTILS/utils`

### Available Functions

#### Validation Orchestrators
* `technical_validations(df, primary_key_columns, critical_columns, range_checks)` - Executes all technical checks
* `combined_validation_result(validation_technical, validation_business)` - Combines validation flags

#### Individual Checks
* `check_not_null(df, columns)` - NULL value checks
* `check_unique(df, columns)` - Duplicate checks
* `check_range(df, column, min_value, max_value)` - Range validation
* `check_referential_integrity(child_df, parent_df, child_col, parent_col)` - FK validation

#### Utility Functions
* `print_validation_header(table_name)` - Formatted headers
* `print_check_result(check_name, status, message, failed_count)` - Formatted results
* `persist_to_delta(df, target_table)` - Delta table persistence

### Usage Pattern
```python
# Load UTILS
%run ../UTILS/utils

# Execute validations
validation_technical, total_rows = technical_validations(
    df=df_result,
    primary_key_columns=["order_id"],
    critical_columns=["order_date", "status"],
    range_checks=[("hour", 0, 23)]
)

# Combine results
validation_passed = combined_validation_result(validation_technical, validation_business)

# Persist if passed
if validation_passed:
    persist_to_delta(df_result, target_table)
```

---

## 🚀 Using the Templates

### Quick Start - Fact/Dimension Tables

1. **Copy template** from `SRC/ETL/TEMPLATES/table_templates`
2. **Open the template** - All cells are documented with instructions
3. **Update Configuration cell** - Set table names, PKs, critical columns
4. **Implement Transformation cells** - Add your business logic
5. **Customize Business Validations** - Add domain-specific rules
6. **Test execution** - Run all cells and verify

### Quick Start - Views

See the dedicated **View Template Guide** at:  
`SRC/ETL/TEMPLATES/VIEW_TEMPLATE_GUIDE.md`

---

## 📏 Code Conventions

### Naming
- **Variables:** `snake_case` (e.g., `df_step1`, `primary_key_columns`)
- **DataFrames:** `df_step1`, `df_result`
- **Tables:** `snake_case` with prefix (e.g., `ft_orders`, `dt_products`)
- **Views:** `snake_case` with prefix `vw_` (e.g., `vw_sales_by_hour`)
- **Notebooks:** Match Delta object name with schema prefix (e.g., `gold_ft_orders`, `gold_vw_sales_by_hour`)
- **Schemas:** `catalog.schema` (e.g., `big_data.bronze`)

### PySpark
- Use `F.` alias for functions
- Chain transformations with `\`
- Prefer DataFrame API over RDD
- Use `.cast()` for type conversions
- Use `.isin()` for multiple value filters

### Style
- All code and comments in **English**
- No emoji in code cells
- No hardcoding - use Configuration variables
- Print clear progress messages

---

## 🏆 Best Practices Summary

### ✅ DO:
- Use the template notebooks as starting point
- Define all names/columns in Configuration
- Separate Technical and Business validations (tables only)
- Persist **only if validations pass** (tables only)
- Use DataFrame API (Serverless compatible)
- Print clear progress messages

### ❌ DON'T:
- Hardcode table/column names
- Mix validation logic in one cell
- Skip validation steps (for tables)
- Use RDD operations
- Use cache/persist on Serverless compute
- Use table structure for simple views

---

## 📞 Support

For questions or improvements:
- Update this document (AGENTS.md)
- Update the template notebooks
- Update VIEW_TEMPLATE_GUIDE.md for view-specific guidance
- Notify the team

---

**Version:** 2.0  
**Last Updated:** 2026-05-31  
**Author:** Data Engineering Team  
**Templates:**
- Fact/Dimension Tables: `SRC/ETL/TEMPLATES/table_templates`
- Views: `SRC/ETL/TEMPLATES/view_template`

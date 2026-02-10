# CLAUDE.md

## Project Overview

This is a **dbt (data build tool)** project called `localdbt`, based on the [Jaffle Shop](https://github.com/clrcrl/jaffle_shop) example. It models a fictional ecommerce store with customers, orders, and payments data. The warehouse is **Apache Spark** with **Delta Lake** table format.

## Repository Structure

```
dbt-projects/
├── profiles.yml              # Spark connection config (Thrift, localhost:10000)
├── requirements.txt          # Python deps: dbt-core 1.8.6, dbt-spark 1.8.0
└── localdbt/                 # Main dbt project
    ├── dbt_project.yml       # Project config (name: localdbt, profile: localdbt)
    ├── models/
    │   ├── staging/          # Source transformations (rename, type cast)
    │   │   ├── stg_customers.sql
    │   │   ├── stg_orders.sql
    │   │   ├── stg_payments.sql
    │   │   └── schema.yml
    │   ├── customers.sql     # Marts: customer aggregation with lifetime value
    │   ├── orders.sql        # Marts: order with pivoted payment methods
    │   ├── schema.yml        # Marts tests and documentation
    │   ├── docs.md           # Doc block for order status enum
    │   ├── overview.md       # Project overview for dbt docs
    │   └── example/          # Starter template models (views)
    ├── macros/               # Empty (placeholder)
    ├── tests/                # Empty (placeholder)
    ├── seeds/                # Empty (placeholder)
    ├── snapshots/            # Empty (placeholder)
    └── analyses/             # Empty (placeholder)
```

## Data Architecture

### Source Data
Raw tables in the `jaffle` schema (referenced directly, no formal `sources.yml`):
- `jaffle.raw_customers` - Customer names
- `jaffle.raw_orders` - Order records
- `jaffle.raw_payments` - Payment records (amounts in cents)

### Model Layers

**Staging (`models/staging/`)** - Clean and rename source columns:
- `stg_customers` - Renames `id` to `customer_id`
- `stg_orders` - Renames `id` to `order_id`, `user_id` to `customer_id`
- `stg_payments` - Renames `id` to `payment_id`, converts amount from cents to dollars

**Marts (`models/`)** - Business logic, materialized as Delta tables:
- `customers` - Joins customer data with aggregated order/payment metrics (first_order, most_recent_order, number_of_orders, customer_lifetime_value)
- `orders` - Joins orders with pivoted payment method amounts using Jinja `for` loop over `['credit_card', 'coupon', 'bank_transfer', 'gift_card']`

### DAG
```
raw_customers -> stg_customers -> customers
raw_orders   -> stg_orders    -> customers, orders
raw_payments -> stg_payments  -> customers, orders
```

## Development Setup

### Prerequisites
- Python 3 with venv
- Apache Spark with Thrift server running on `localhost:10000`

### Environment Setup
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### Verify Connection
```bash
dbt debug --profiles-dir ./ --project-dir ./localdbt
```

## Common dbt Commands

All commands should be run from the repo root, pointing to the correct directories:

```bash
# Run all models
dbt run --profiles-dir ./ --project-dir ./localdbt

# Run a specific model
dbt run --profiles-dir ./ --project-dir ./localdbt --select customers

# Run staging models only
dbt run --profiles-dir ./ --project-dir ./localdbt --select staging.*

# Run tests
dbt test --profiles-dir ./ --project-dir ./localdbt

# Test a specific model
dbt test --profiles-dir ./ --project-dir ./localdbt --select customers

# Generate and serve docs
dbt docs generate --profiles-dir ./ --project-dir ./localdbt
dbt docs serve --profiles-dir ./ --project-dir ./localdbt

# Clean compiled artifacts
dbt clean --profiles-dir ./ --project-dir ./localdbt
```

## Warehouse Configuration

- **Type:** Apache Spark
- **Connection:** Thrift protocol
- **Host/Port:** localhost:10000
- **Schema:** `jaffle`
- **Table format:** Delta Lake (marts models use `file_format='delta'`)
- **Threads:** 1
- **Target:** `dev`

## Testing Conventions

Tests are defined in `schema.yml` files alongside models (not as singular SQL tests):

| Test Type | Usage |
|-----------|-------|
| `unique` | All primary keys (customer_id, order_id, payment_id) |
| `not_null` | All primary keys and critical measure columns |
| `accepted_values` | Enum columns (status, payment_method) |
| `relationships` | Foreign keys (orders.customer_id -> customers.customer_id) |

## Code Conventions

- **SQL style:** Lowercase keywords, CTEs with descriptive names, final CTE pattern (`with ... final as (...) select * from final`)
- **Materialization:** Staging models use default (view); marts models use `table` with `file_format='delta'` via `{{ config() }}`
- **Naming:** Staging models prefixed with `stg_`; source column renames happen in staging layer
- **References:** Use `{{ ref('model_name') }}` for model dependencies; raw source tables referenced directly as `jaffle.raw_*`
- **Jinja:** Used for dynamic SQL generation (e.g., payment method pivoting in `orders.sql`) and doc references (`{{ doc("orders_status") }}`)
- **No external packages:** The project has no `packages.yml`; all logic is self-contained
- **Amount convention:** Raw amounts are in cents; staging layer converts to dollars (divide by 100)
- **Currency:** AUD (Australian Dollars)

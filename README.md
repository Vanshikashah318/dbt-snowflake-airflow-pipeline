# dbt + Snowflake ELT Pipeline Orchestrated with Airflow

An end-to-end ELT pipeline that transforms Snowflake's TPC-H sample data into an analytics-ready orders fact table. It uses **dbt** for modeling and testing, and **Apache Airflow** (Astronomer Runtime + Astronomer Cosmos) for scheduling and orchestration.

## Architecture

```
Snowflake (snowflake_sample_data.tpch_sf1)
        │  orders, lineitem
        ▼
 staging (views)          staging_tpch_orders, staging_tpch_line_items
        │
        ▼
 intermediate (tables)    int_order_items → int_order_items_summary
        │
        ▼
 marts (tables)           fct_orders
        │
        ▼
 Airflow (Cosmos)         one task per dbt model + test, scheduled daily
```

Cosmos reads the dbt project and turns every model and its tests into its own Airflow task. Dependencies come from dbt's `ref()` graph, so failures and retries happen per model rather than for the whole `dbt run`.

## Tech stack

- **Snowflake**: data warehouse (source data and transformed models)
- **dbt Core** (`dbt-snowflake`): SQL transformations, tests, macros
- **Apache Airflow 3** on **Astronomer Runtime**: orchestration, run locally with the Astro CLI and Docker
- **Astronomer Cosmos**: renders the dbt project as an Airflow DAG
- **dbt_utils**: surrogate key generation

## dbt project highlights

| Layer | Model | What it does |
|---|---|---|
| Staging | `staging_tpch_orders` | Renames and selects order columns from the source |
| Staging | `staging_tpch_line_items` | Builds a surrogate key (`dbt_utils.generate_surrogate_key`) on order + line number |
| Intermediate | `int_order_items` | Joins orders to line items and computes item discount amounts |
| Intermediate | `int_order_items_summary` | Aggregates gross sales and discounts per order |
| Mart | `fct_orders` | Order-level fact table with sales and discount metrics |

- **Reusable macro**: `discounted_amount()` in `macros/pricing.sql` centralizes the discount calculation.
- **Data quality tests**:
  - generic: `unique`, `not_null`, `relationships`, `accepted_values` on the source and the mart
  - singular: `tests/fct_orders_discount.sql`, which flags orders with a positive discount amount; discounts are stored as negative values
- **Materialization strategy**: staging models are views; intermediate and mart models are tables, run on a dedicated warehouse (`dbt_wh`).

## Project structure

```
├── dags/
│   ├── dbt_dag.py                 # Cosmos DbtDag definition (daily schedule)
│   └── dbt/data_pipelie/          # dbt project (models, macros, tests)
├── Dockerfile                     # Astro Runtime + isolated dbt virtualenv
├── requirements.txt               # astronomer-cosmos, Snowflake provider
└── airflow_settings.yaml          # local-only connections (gitignored)
```

dbt is installed in its own virtualenv inside the image (`dbt_venv`), so its dependencies don't conflict with Airflow's.

## Running locally

**Prerequisites:** Docker Desktop, the [Astro CLI](https://www.astronomer.io/docs/astro/cli/install-cli), and a Snowflake account with:
- a warehouse `dbt_wh`
- a database `dbt_db` with schema `dbt_schema`
- a role `dbt_role` that can use all three

1. Start Airflow:
   ```bash
   astro dev start
   ```
2. In the Airflow UI, go to **Admin → Connections** and add a Snowflake connection:

   | Field | Value |
   |---|---|
   | Connection Id | `snowflake_conn` |
   | Login / Password | your Snowflake credentials |
   | Account | your account identifier, e.g. `xy12345.us-east-2.aws` |
   | Warehouse / Database / Role | `dbt_wh` / `dbt_db` / `dbt_role` |

3. Unpause and trigger `dbt_dag`.

Credentials stay in Airflow's connection store and are never committed. Cosmos generates the dbt `profiles.yml` from the connection at runtime.

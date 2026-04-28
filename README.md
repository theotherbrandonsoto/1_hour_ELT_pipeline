# ELT_pipeline

A end-to-end ELT pipeline built with dbt, Snowflake, and Apache Airflow, as a demonstration of modern data stack fundamentals.

Built with assistance from Claude.

## What it does

Raw order and line item data from Snowflake's built-in TPCH sample dataset is transformed through a layered dbt project and orchestrated by Airflow via Astronomer Cosmos.

It simulates a wholesale supplier business and includes tables for orders, line items, customers, parts, suppliers, and more. The data is fictional but designed to mimic realistic business relationships and query patterns.

The pipeline is scheduled to run daily and produces a `fct_orders` table joining order-level data with aggregated line item financials including gross sales and discount amounts. Designed for local development -- production deployment would use a hosted Airflow service such as Astronomer Cloud or AWS MWAA.

## Stack

- **Snowflake** -- cloud data warehouse and source of raw TPCH data
- **dbt Core** -- transformation layer with staging views, intermediate tables, and a facts mart
- **Apache Airflow** -- orchestration, running via Astronomer's Astro CLI
- **Astronomer Cosmos** -- renders dbt models as individual Airflow task nodes

## Project structure

```
ELT_Pipe/
├── data_pipeline/               # dbt project
│   ├── models/
│   │   ├── staging/             # views over raw source tables
│   │   │   ├── stg_tpch_orders.sql
│   │   │   ├── stg_tpch_line_items.sql
│   │   │   └── tpch_sources.yml
│   │   └── marts/               # intermediate and fact tables
│   │       ├── int_order_items.sql
│   │       ├── int_order_items_summary.sql
│   │       ├── fct_orders.sql
│   │       └── generic_tests.yml
│   ├── macros/
│   │   └── pricing.sql          # reusable discounted_amount macro
│   ├── tests/                   # custom singular tests
│   ├── packages.yml             # dbt_utils dependency
│   └── dbt_project.yml
└── dbt_dag/                     # Astro/Airflow project
    ├── dags/
    │   ├── dbt_dag.py           # DbtDag definition using Cosmos
    │   └── dbt/
    │       └── data_pipeline/   # dbt project mounted for Airflow
    ├── Dockerfile
    └── requirements.txt
```

## Data model

```
snowflake_sample_data.tpch_sf1
    └── orders          (raw)
    └── lineitem        (raw)
          |
    stg_tpch_orders     (staging view)
    stg_tpch_line_items (staging view)
          |
    int_order_items          (intermediate table)
    int_order_items_summary  (intermediate table)
          |
    fct_orders          (facts mart table)
```

## Setup

### Prerequisites

- Snowflake account with `ACCOUNTADMIN` or equivalent role
- Python 3.10+
- Astro CLI installed

### dbt

1. Create a virtual environment and install dependencies:
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   pip install dbt-core dbt-snowflake
   ```

2. Configure `~/.dbt/profiles.yml` with your Snowflake connection. Keypair authentication is recommended:
   ```yaml
   data_pipeline:
     outputs:
       dev:
         type: snowflake
         account: <your_account>
         user: <your_user>
         private_key_path: ~/.ssh/snowflake_key.p8
         authenticator: snowflake_jwt
         database: dbt_db
         schema: dbt_schema
         warehouse: dbt_wh
         role: dbt_role
         threads: 10
     target: dev
   ```

3. Install dbt packages and verify the connection:
   ```bash
   cd data_pipeline
   dbt deps
   dbt debug
   ```

4. Run the models:
   ```bash
   dbt run
   dbt test
   ```

### Airflow

1. Start the Astro environment:
   ```bash
   cd dbt_dag
   astro dev start
   ```

2. Open the Airflow UI at `http://localhost:8080` and create a Snowflake connection under **Admin > Connections** with connection ID `snowflake_conn`.

3. Trigger the `dbt_dag` DAG.

## Notes

- `profiles.yml` and private keys are intentionally excluded from this repo and should be configured locally
- The Airflow connection stores Snowflake credentials and is configured through the UI, not committed to source control

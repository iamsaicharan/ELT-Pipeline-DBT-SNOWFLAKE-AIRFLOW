# ELT Pipeline

Tech stack used here is DBT for transformation, Snowflake for Data warehousing and Airflow for Orchesetration.

## Setup dbt and Snowflake

Install dbt core by using pip

```bash
pip install dbt-core dbt-snowflake
```

Setup the snowflake using the `snowflake.txt` file

Start the dbt project using

```bash
dbt init
```

and give it a name `elt_pipeline` and select snowflake and signin using locator, username, password, role, warehouse, database, schema and threads

this will create elt_pipeline project folder

## Setup dbt_project.yml file

This is the main file that dbt uses ffor understanding the project. In the models attribute we are going to create two tables - staging and marts.

```yaml
models:
  elt_pipeline:
    # Config indicated by + and applies to all files under models/example/
    staging:
      +materialized: view
      snowflake_warehouse: dbt_wh
    marts:
      +materialized: table
      snowflake_warehouse: dbt_wh
```

Note: Delete the example models folder in the models folders and create two folders naming `staging` and `marts`.

Now lets install third party packages.

create a `packages.yaml` file in the `elt_pipeline` folder.

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
```

Now run the deps

```bash
dbt deps
```
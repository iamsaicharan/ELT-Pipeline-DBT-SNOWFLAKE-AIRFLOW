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


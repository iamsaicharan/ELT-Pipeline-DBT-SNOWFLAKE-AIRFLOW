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

and give it a name `data_pipeline` and select snowflake and signin using locator, username, password, role, warehouse, database, schema and threads

this will create elt_pipeline project folder

## Setup dbt_project.yml file

This is the main file that dbt uses ffor understanding the project. In the models attribute we are going to create two tables - staging and marts.

```yml
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

create a `packages.yml` file in the `elt_pipeline` folder.

```yml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
```

Now run the deps

```bash
dbt deps
```

## Create source and staging tables

create a `tpch_sources.yml` file in the staging folder.

```yml
version: 2

sources:
  - name: tpch
    database: snowflake_sample_data
    schema: tpch_sf1
    tables:
      - name: orders
        columns:
          - name: o_orderkey
            tests:
              - unique
              - not_null
      - name: lineitem
        columns:
          - name: l_orderkey
            tests:
              - relationships:
                  to: source('tpch', 'orders')
                  field: o_orderkey
```

create a `stg_tpch_orders.sql` file in the staging folder.

```sql
select
    o_orderkey as order_key,
    o_custkey as customer_key,
    o_orderstatus as status_code,
    o_totalprice as total_price,
    o_orderdate as order_date
from
    {{ source('tpch', 'orders') }}
```

create a `stg_tpch_line_items.sql` file in the staging folder.

```sql
select
    {{
        dbt_utils.generate_surrogate_key([
            'l_orderkey',
            'l_linenumber'
        ])
    }} as order_item_key,
   l_orderkey as order_key,
   l_partkey as part_key,
    l_linenumber as line_number,
    l_quantity as quantity,
    l_extendedprice as extended_price,
    l_discount as discount_percentage,
    l_tax as tax_rate
from
    {{ source('tpch', 'lineitem') }}
```

run dbt this will create tables

```bash
dbt run
```

## Transform models

create a intermediate table `int_order_items.sql` in the marts folder

```sql
select
    line_item.order_item_key,
    line_item.part_key,
    line_item.line_number,
    line_item.extended_price,
    orders.order_key,
    orders.customer_key,
    orders.order_date,
    {{ discounted_amount('line_item.extended_price', 'line_item.discount_percentage') }} as item_discount_amount
from
    {{ ref('stg_tpch_orders') }} as orders
join
    {{ ref('stg_tpch_line_items') }} as line_item
        on orders.order_key = line_item.order_key
order by
    orders.order_date
```

## Create Macros

create `pricing.sql`

```sql
{% macro discounted_amount(extended_price, discount_percentage, scale=2) %}
    (-1 * {{extended_price}} * {{discount_percentage}})::decimal(16, {{ scale }})
{% endmacro %}
```

## More transform models

create `int_order_items_summary.sql` and `fct_orders.sql`

```sql
select 
    order_key,
    sum(extended_price) as gross_item_sales_amount,
    sum(item_discount_amount) as item_discount_amount
from
    {{ ref('int_order_items') }}
group by
    order_key
```

```sql
select
    orders.*,
    order_item_summary.gross_item_sales_amount,
    order_item_summary.item_discount_amount
from
    {{ref('stg_tpch_orders')}} as orders
join
    {{ref('int_order_items_summary')}} as order_item_summary
        on orders.order_key = order_item_summary.order_key
order by order_date
```

rub the dbt.

## Generic and Singular tests

Create `generic_tests.yml` in the marts

```yml
models:
  - name: fct_orders
    columns:
      - name: order_key
        tests:
          - unique
          - not_null
          - relationships:
              to: ref('stg_tpch_orders')
              field: order_key
              severity: warn
      - name: status_code
        tests:
          - accepted_values:
              values: ['P', 'O', 'F']
```

 Create singular tests `fct_orders_discount.sql` in tests folder

 ```sql
 select
    *
from
    {{ref('fct_orders')}}
where
    item_discount_amount > 0
 ```

 create `fct_orders_date_valid.sql` in tests folder

 ```sql
 select
    *
from
    {{ref('fct_orders')}}
where
    date(order_date) > CURRENT_DATE()
    or date(order_date) < date('1990-01-01')
 ```

run dbt test

```bash
dbt test
```

## Deploy the dags in AirFlow

Create a folder (dbt-dag) in the root and install the below

Update requirements.txt

```terminal
astronomer-cosmos
apache-airflow-providers-snowflake
```

cd into the dbt-dag folder and create a astro dev init

```bash
astro dev init
```

this will create a new astro project, add the dockerfile update, update requirements

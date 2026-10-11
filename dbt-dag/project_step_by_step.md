# Code along project - Build ELT Pipeline with dbt, Snowflake and Airflow

## Step 1: Setup dbt + Snowflake

- Tạo tài khoản Snowflake trên web + cài đặt dbt OSS ngay trên máy.
- Set up warehouse dbt_wh, database dbt_db và schema dbt_schema trên Snowflake.
- Tạo thư mục dbt project ở ngay trên máy tính (dùng dbt init data_pipeline để tạo project; git init để bắt đầu quản lý phiên bản bằng Git).
  --> Nó sẽ lưu vào máy một thư mục project gồm các file để mình viết và chạy dbt.

## Step 2: Config dbt_project.yml and packages

- Config các file trong project sẽ trả về dạng view hay table trong dbt_project.yml.

```yaml
models:
  data_pipeline:
    staging:
      +materialized: view
      +snowflake_warehouse: dbt_wh
    marts:
      +materialized: table
      +snowflake_warehouse: dbt_wh
```

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.4.1
```

- Những file ở staging sẽ trả về dạng view; những file ở marts sẽ trả về dạng table.
- +snowflake_warehouse: dbt_wh: Chỉ định virtual warehouse dbt_wh dùng tài nguyên tính toán cho model.
- Ở file packages.yml, khai báo package dbt_utils để dùng generate_surrogate_key; sau đó chạy dbt deps để cài package.

## Step 3: Create source and staging tables

- Tạo file YAML để config source kèm theo test null, unique, relationships --> Ta sẽ tạo 1 folder mới ở trong models tên là staging --> Sau đó tạo 1 file tpch_source.yml (nó sẽ nằm trong models/staging) và ở file này sẽ là nơi mình config source và tạo các generic test.

```yaml
version: 2

sources:
  - name: tpch # Đặt tên cho source
    database: snowflake_sample_data # Nơi chứa source
    schema: tpch_sf1 # Schema chứa hai bảng nguồn Orders và Lineitem
    tables:
      - name: orders # Test cột mã đơn hàng không null và không trùng
        columns:
          - name: o_orderkey
            tests:
              - not_null
              - unique

      - name: lineitem # Test relationship: mỗi l_orderkey khác null trong lineitem phải tồn tại ở trong orders
        columns:
          - name: l_orderkey
            tests:
              - relationships:
                  arguments:
                    to: source('tpch', 'orders')
                    field: o_orderkey
```

- Tạo các file SQL để đổi lại tên sao cho phù hợp.

### Tạo stg_tpch_line_items.sql ở trong models/staging.

Surrogate key có nghĩa là cứ mỗi 1 cặp order_key và line number sẽ xác định một dòng hàng trong đơn hàng.
Do ở bên trên ta đã config source nên có thể dùng hàm source ở đây.

```sql
SELECT
    {{
        dbt_utils.generate_surrogate_key([
            'l_orderkey',
            'l_linenumber'
        ])
    }} AS order_item_key,
    l_orderkey AS order_key,
    l_partkey AS part_key,
    l_linenumber AS line_number,
    l_quantity AS quantity,
    l_extendedprice AS extended_price,
    l_discount AS discount_percentage,
    l_tax AS tax_rate
FROM
    {{source('tpch', 'lineitem')}}
```

### Tạo stg_tpch_orders.sql.

```sql
SELECT
    o_orderkey AS order_key,
    o_custkey AS customer_key,
    o_orderstatus AS status_code,
    o_totalprice AS total_price,
    o_orderdate AS order_date
FROM
    {{source('tpch', 'orders')}}
```

 --> Chạy dbt run để update tables or views trên data warehouse. Nếu muốn chạy cả các test thì dùng dbt test sau đó hoặc dbt build.

## Step 4: Transformed model (fact table, data marts)

- Create intermediate table (Join order and line_item table).
- Ta tạo thêm marts folder ở trong models (models/marts).

### models/marts/int_order_items.sql

Với những bảng mà ta đã transform trước đó ở trong stg thì ta có thể dùng ref để gọi luôn.

```sql
SELECT
    line_item.order_item_key,
    line_item.part_key,
    line_item.line_number,
    line_item.extended_price,
    orders.order_key,
    orders.customer_key,
    orders.order_date
FROM
    {{ref('stg_tpch_orders')}} AS orders
JOIN
    {{ref('stg_tpch_line_items')}} AS line_item
        ON orders.order_key = line_item.order_key
ORDER BY
    orders.order_date
```

## Step 5: Create Macros

- Ta sẽ tạo các macros ví dụ như tính discount amount và dùng nó ở những SQL khác mà không cần phải viết lại --> Nếu muốn thay đổi gì ta chỉ cần thay đổi ở ngay trong macro.

### Ta sẽ tạo 1 file ví dụ như pricing.sql trong macros folder (macros/pricing.sql).

```sql
{% macro discount_amount(extended_price, discount_percentage, scale=2) %}
    (-1 * {{ extended_price }} * {{ discount_percentage }})::numeric(16, {{ scale }})
{% endmacro %}
```

--> Hàm này sẽ giúp ta tính được số tiền discount (* -1 để thể hiện khoản giảm giá là số âm).

**Một số ứng dụng của hàm macros:**
- Ở models/marts/int_order_items.sql, ta có thể thêm cột discount_amount bằng cách sử dụng hàm macros vừa làm. Gọi lại hàm mà không cần phải viết lại công thức.

```sql
SELECT
    line_item.order_item_key,
    line_item.part_key,
    line_item.line_number,
    line_item.extended_price,
    orders.order_key,
    orders.customer_key,
    orders.order_date,
    {{ discount_amount('line_item.extended_price', 'line_item.discount_percentage') }} AS item_discount_amount
FROM
    {{ref('stg_tpch_orders')}} AS orders
JOIN
    {{ref('stg_tpch_line_items')}} AS line_item
        ON orders.order_key = line_item.order_key
ORDER BY
    orders.order_date
```

### models/marts/int_order_items_summary.sql

```sql
SELECT
    order_key,
    sum(extended_price) AS gross_item_sales_amount,
    sum(item_discount_amount) AS item_discount_amount
FROM
    {{ref('int_order_items')}}
GROUP BY
    order_key
```

--> Tính tổng theo từng đơn đặt hàng.

### models/marts/fct_orders.sql: fact table là bảng chứa số liệu theo từng đơn hàng để mình tính toán, phân tích.

```sql
SELECT
    orders.*,
    order_item_summary.gross_item_sales_amount,
    order_item_summary.item_discount_amount
FROM
    {{ref('stg_tpch_orders')}} AS orders
JOIN
    {{ref('int_order_items_summary')}} AS order_item_summary
        ON orders.order_key = order_item_summary.order_key
ORDER BY
    orders.order_date
```

## Step 6: Generic and Singular tests

- Tạo các test.

### generic test (models/marts/generic_tests.yml)

```yaml
version: 2

models:
  - name: fct_orders
    columns:
      - name: order_key
        tests:
          - not_null
          - unique
          - relationships:
              arguments:
                to: ref('stg_tpch_orders')
                field: order_key
              config:
                severity: warn
      - name: status_code
        tests:
          - accepted_values:
              arguments:
                values: ['F', 'O', 'P']
```

- Test accepted_values kiểm tra những giá trị cụ thể được cho phép trong cột status_code; nếu có giá trị khác test sẽ fail.
- severity: warn chỉ áp dụng cho test relationships bên trên: nếu có giá trị không khớp thì sẽ hiện warning, không làm test đó báo error.

### singular test: Mỗi file SQL bên dưới trả về các dòng bị lỗi; nếu trả về 0 dòng thì test pass.

### tests/fct_orders_date_valid.sql

```sql
SELECT
    *
FROM
    {{ref('fct_orders')}}
WHERE
    date(order_date) > CURRENT_DATE()
    OR date(order_date) < date('1990-01-01')
```

### tests/fct_orders_discount.sql

```sql
SELECT
    *
FROM
    {{ref('fct_orders')}}
WHERE
    item_discount_amount > 0
```

## Step 7: Deploy models using Airflow

- Cài đặt Docker và Cosmos để chạy trong môi trường local.
- Docker là một tool giúp ta đóng gói project lại, chuẩn hóa các phiên bản và dependencies để tránh bị crash tính năng giữa các môi trường khác nhau.
- Airflow là dự án Apache; Astronomer phát triển Astro CLI và Cosmos để hỗ trợ làm việc với Airflow. Cosmos là một thư viện tích hợp dbt vào Airflow.
--> Docker chạy môi trường → Astro CLI giúp khởi động Airflow → Airflow thực hiện workflow trong DAG → Cosmos gọi các task dbt → dbt gửi SQL cho Snowflake xử lý.

### Tạo file dags/dbt_dag.py:

```python
import os
from datetime import datetime

from cosmos import DbtDag, ProjectConfig, ProfileConfig, ExecutionConfig
from cosmos.profiles import SnowflakeUserPasswordProfileMapping

profile_config = ProfileConfig(
    profile_name="data_pipeline",
    target_name="dev",
    profile_mapping=SnowflakeUserPasswordProfileMapping(
        conn_id="snowflake_conn", 
        profile_args={"database": "dbt_db", "schema": "dbt_schema"},
    )
)

dbt_snowflake_dag = DbtDag(
    project_config=ProjectConfig("/usr/local/airflow/dags/dbt/data_pipeline",),
    operator_args={"install_deps": True},
    profile_config=profile_config,
    execution_config=ExecutionConfig(dbt_executable_path=f"{os.environ['AIRFLOW_HOME']}/dbt_venv/bin/dbt",),
    schedule="@daily",
    start_date=datetime(2023, 9, 10),
    catchup=False,
    dag_id="dbt_dag",
)
```

--> File này sẽ giúp ta config được lịch trình, những thứ cần chạy, workflow chạy sao cho hợp lý trên Airflow.

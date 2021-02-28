# Coffee Shop ETL and Reporting

## Project Details

Designing the etl process of the data of a coffee chain and generating analyzes and dashboard based on summary data.

## Resource

https://www.kaggle.com/ylchang/coffee-shop-sample-data-1113

## Tools

- In the project, Google Cloud Composer, Apache Airflow, Python and BigQuer were used to operate the ETL process.
- BigQuery was used for analysis.
- Based on the summary data, a continuously trackable dashboard was created. PowerBI is used for this.

## Steps

### Step 1 : Creating a Bucket in Google Cloud Storage

![Cloud Storage Bucket](https://user-images.githubusercontent.com/66178028/109412015-c3262d80-79b6-11eb-9261-8885c40bc711.png)

### Step 2 : Uploading Files to Google Cloud Storage 

![Uploading Files](https://user-images.githubusercontent.com/66178028/109412078-041e4200-79b7-11eb-9839-801f51315284.png)

### Step 3 : Creating OLTP tables in Google Cloud BigQuery

I wanted to simulate the csv data as if it was actually coming from an OLTP system. For this, I pulled the csv data from Storage to BigQuery

![Ekran Resmi 2021-02-28 11 23 39](https://user-images.githubusercontent.com/66178028/109412163-6f681400-79b7-11eb-8d4c-bc503a4708eb.png)

![Ekran Resmi 2021-02-28 11 24 24](https://user-images.githubusercontent.com/66178028/109412182-8f97d300-79b7-11eb-98f9-d540f8dc4767.png)

### Step 4 : Creating Google Cloud Composer & Airflow Environment

I built the ETL process with Python. I will be able to run scheduling and monitoring steps with the help of Composer & Airflow.

![Ekran Resmi 2021-02-28 11 26 43](https://user-images.githubusercontent.com/66178028/109412237-db4a7c80-79b7-11eb-95d1-40cd4a71351e.png)

### Step 5 : Creating first DAG with Python

I created Airflow DAG with Python. Tables are created in DWH scheme by pulling data from OLTP tables and doing some transformations.

While doing this, I combined some tables, simplified them and added new fields for enrichment.

The system works by finding the maximum day to receive new incoming sales. However, I performed the initial load in order to get all sales first.

```
from datetime import timedelta, datetime
import json
from airflow import DAG
from airflow.contrib.operators.bigquery_operator import BigQueryOperator
from airflow.operators.bash_operator import BashOperator
from airflow import models
from airflow.utils.dates import days_ago



default_args = {
    'owner': 'onurokyol',
    'depends_on_past': False,
    'start_date': days_ago(1),
    'email': ['onurokyol14@gmail.com'],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 0,
    'retry_delay': timedelta(minutes=5)
}

schedule_interval = "30 6 * * *"

dag_etl = DAG(
    'dwhpackage',
    default_args=default_args,
    schedule_interval=schedule_interval,
)



BQ_CONN_ID = "bigquery_default"

customer_etl = BigQueryOperator(
task_id = 'customer_etl',
bigquery_conn_id = BQ_CONN_ID,
use_legacy_sql=False,
bql = '''
#standartSQL
CREATE OR REPLACE TABLE DWH.Customer AS
SELECT 
d1.*,
d2.generation,
DATE_DIFF(CURRENT_DATE(),d1.birthdate,YEAR)  AS Age,
DATE_DIFF(CURRENT_DATE(),d1.customer_since,YEAR) AS MembershipYear
FROM OLTP.customer d1
LEFT JOIN OLTP.generations d2 ON d1.birth_year = d2.birth_year
''',
dag = dag_etl
)


date_etl = BigQueryOperator(
task_id = 'date_etl',
bigquery_conn_id = BQ_CONN_ID,
use_legacy_sql=False,
bql = '''
#standartSQL
CREATE OR REPLACE TABLE DWH.Date AS
SELECT *,EXTRACT(DAY FROM transaction_date) Day_ID,FORMAT_DATE("%A",transaction_date) as Day_Desc FROM OLTP.dates
''',
dag = dag_etl
)

product_etl = BigQueryOperator(
task_id = 'product_etl',
bigquery_conn_id = BQ_CONN_ID,
use_legacy_sql=False,
bql = '''
#standartSQL
CREATE OR REPLACE TABLE DWH.Product AS
select * from OLTP.product
''',
dag = dag_etl
)

store_etl = BigQueryOperator(
task_id = 'store_etl',
bigquery_conn_id = BQ_CONN_ID,
use_legacy_sql=False,
bql = '''
#standartSQL
CREATE OR REPLACE TABLE DWH.Store AS
select d1.*,d2.First_Name || ' ' || d2.Last_Name AS Store_Manager_Full_Name from OLTP.sales_outlet d1
left join OLTP.staff d2 on d1.manager = d2.staff_id
''',
dag = dag_etl
)

staff_etl = BigQueryOperator(
task_id = 'staff_etl',
bigquery_conn_id = BQ_CONN_ID,
use_legacy_sql=False,
bql = '''
#standartSQL
CREATE OR REPLACE TABLE DWH.Staff AS
SELECT d1.staff_id,d1.first_name || ' ' || d1.last_name AS full_name,d1.position,d1.start_date,
CASE WHEN d2.sales_outlet_id is NOT NULL THEN 'Store' || ' ' || store_state_province ELSE d1.location END AS location FROM OLTP.staff d1
left join OLTP.sales_outlet d2 on d1.Location = CAST(d2.sales_outlet_id as string)
''',
dag = dag_etl
)

salestarget_etl = BigQueryOperator(
task_id = 'salestarget_etl',
bigquery_conn_id = BQ_CONN_ID,
use_legacy_sql=False,
bql = '''
#standartSQL
CREATE OR REPLACE TABLE DWH.SalesTarget AS
SELECT
sales_outlet_id,
'20' || SUBSTR(year_month,5,6) ||
CASE
when upper(SUBSTR(year_month,0,3)) = 'JAN' then '01' 
when upper(SUBSTR(year_month,0,3)) = 'FEB' then '02' 
when upper(SUBSTR(year_month,0,3)) = 'MAR' then '03' 
when upper(SUBSTR(year_month,0,3)) = 'APR' then '04' 
when upper(SUBSTR(year_month,0,3)) = 'MAY' then '05' 
when upper(SUBSTR(year_month,0,3)) = 'JUN' then '06' 
when upper(SUBSTR(year_month,0,3)) = 'JUL' then '07' 
when upper(SUBSTR(year_month,0,3)) = 'AUG' then '08' 
when upper(SUBSTR(year_month,0,3)) = 'SEP' then '09' 
when upper(SUBSTR(year_month,0,3)) = 'OCT' then '10' 
when upper(SUBSTR(year_month,0,3)) = 'NOV' then '11' 
when upper(SUBSTR(year_month,0,3)) = 'DEC' then '12' 
END AS Year_Month,
beans_goal,
beverage_goal,
food_goal,
merchandise__goal AS merchandise_goal,
total_goal
FROM OLTP.sales_targets
''',
dag = dag_etl
)

pastryinventory_etl = BigQueryOperator(
task_id = 'pastryinventory_etl',
bigquery_conn_id = BQ_CONN_ID,
use_legacy_sql=False,
bql = '''
#standartSQL
CREATE OR REPLACE TABLE DWH.PastryInventory AS
select sales_outlet_id,
transaction_date,
product_id,
start_of_day,
quantity_sold,
start_of_day - quantity_sold AS waste
from OLTP.pastry_inventory
''',
dag = dag_etl
)

salessummary_etl = BigQueryOperator(
task_id = 'salessummary_etl',
bigquery_conn_id = BQ_CONN_ID,
use_legacy_sql=False,
bql = '''
#standartSQL
INSERT INTO DWH.SalesSummary
SELECT transaction_date,sales_outlet_id,staff_id,customer_id,product_id,
SUM(quantity) AS quantity,
SUM(line_item_amount)  line_item_amount
FROM OLTP.sales_receipt
WHERE transaction_date > (SELECT MAX(transaction_date) FROM DWH.SalesSummary)
GROUP BY transaction_date,sales_outlet_id,staff_id,customer_id,product_id
''',
dag = dag_etl
)

customer_etl >> date_etl >> product_etl >> store_etl >> staff_etl >> salestarget_etl >> pastryinventory_etl >> salessummary_etl
```

### Step 6 : Uploading .py DAG file to Cloud Composer DAG Folder

![Ekran Resmi 2021-02-28 11 36 33](https://user-images.githubusercontent.com/66178028/109412475-3f217500-79b9-11eb-824c-749e8fc409c9.png)

### Step 6 : Trigger First Airflow DAG

DAG was triggered and all its steps worked successfully. After that, DAG will run at 6:30 each morning and summarize all the tables to DWH.

![Ekran Resmi 2021-02-28 11 38 08](https://user-images.githubusercontent.com/66178028/109412518-7db72f80-79b9-11eb-971a-accb7a8a7592.png)

Summary tables successfully created in BigQuery DWH scheme.

![Ekran Resmi 2021-02-28 11 40 10](https://user-images.githubusercontent.com/66178028/109412573-d5559b00-79b9-11eb-91d8-a51d70919297.png)


![PoweBI Star Schema](https://user-images.githubusercontent.com/66178028/109411752-50688280-79b5-11eb-9f93-6c35382a96ca.PNG)



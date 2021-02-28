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

![PoweBI Star Schema](https://user-images.githubusercontent.com/66178028/109411752-50688280-79b5-11eb-9f93-6c35382a96ca.PNG)



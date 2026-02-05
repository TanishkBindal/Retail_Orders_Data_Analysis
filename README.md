# Retail Orders ETL and Analysis

## 📋 Project Overview
This project demonstrates an end-to-end data analysis pipeline on the Retail Orders dataset using Python and SQL Server. Data is sourced via the Kaggle API, cleaned and processed using Python (Pandas), and then loaded into SQL Server for analysis. Advanced SQL concepts such as CTEs, subqueries, and window functions are used to extract meaningful business insights, showcasing a practical and scalable analytics workflow.

## Table of Contents
- [ Tools and Technologies Used](#tools-and-technologies-used)
- [Getting Started](#getting-started)
- [ETL Process](#etl-process)
- [Data Analysis](#data-analysis)
- [Queries](#queries)
- [Conclusion](#conclusion)

## Tools and Technologies Used
![Kaggle](https://img.shields.io/badge/Kaggle-035a7d?style=for-the-badge&logo=kaggle&logoColor=white)&nbsp;
![Jupyter Notebook](https://img.shields.io/badge/jupyter-%23FA0F00.svg?style=for-the-badge&logo=jupyter&logoColor=white)&nbsp;
![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=yellow)&nbsp;
![Pandas](https://img.shields.io/badge/pandas-%23150458.svg?style=for-the-badge&logo=pandas&logoColor=white)&nbsp;
![Microsoft SQL Server](https://img.shields.io/badge/Microsoft%20SQL%20Server-CC2927?style=for-the-badge&logo=microsoft-sql-server&logoColor=white)&nbsp; 
![MySQL](https://img.shields.io/badge/sql-4479A1.svg?style=for-the-badge&logo=sql&logoColor=white)&nbsp; 

## Getting Started

### Prerequisites

- Python 
- SQLServer
- Kaggle API
- Required Python packages:
  - pandas
  - sqlalchemy
  - kaggle


1. **Install the required Python packages:**

   ```sh
   pip install pandas sqlalchemy kaggle
   ```

2. **Set up Kaggle API:**

   - Place your `kaggle.json` file (download from kaggle website) in `~/.kaggle/`.

## ETL Process

<div align= "center">
 <img src="https://github.com/TanishkBindal/Retail_Orders_Data_Analysis/blob/main/ETL_Pipeline.png?raw=true"
   width= "700" height="700"/>
</div>

### 1. Import and Authenticate Kaggle API

```python
import kaggle
kaggle.api.authenticate()
```

### 2. Download Dataset

```python
kaggle.api.dataset_download_files('ankitbansal06/retail-orders', unzip=True)
```

### 3. Read Data from File using Pandas

```python
import pandas as pd
df = pd.read_csv('orders.csv')
df.head(20)
```

### 4. Data Cleaning

- Replace void entries (e.g., 'unknown', 'not available') with NULL:

  ```python
  df = pd.read_csv('orders.csv', na_values=['Not Available', 'unknown'])
  ```

- Rename columns to be code-friendly:

  ```python
  df.columns = df.columns.str.lower().str.replace(' ', '_')
  ```

### 5. Data Manipulation

- Add new columns:

  ```python
  df['discount'] = df['list_price'] * df['discount_percent'] / 100
  df['sale_price'] = df['list_price'] - df['discount']
  df['profit'] = round(df['sale_price'] - df['cost_price'], 2)
  ```

- Drop unnecessary columns:

  ```python
  df.drop(columns=['list_price', 'cost_price', 'discount_percent'], inplace=True)
  ```

- Convert `order_date` to datetime:

  ```python
  df['order_date'] = pd.to_datetime(df['order_date'], format='%Y-%m-%d')
  ```

### 6. Connect to Database

```python
import sqlalchemy as sal
engine = sal.create_engine(r'mssql://LAPTOP-LSJUB7F3\SQLEXPRESS/master?driver=ODBC+DRIVER+17+FOR+SQL+SERVER')
conn = engine.connect()
```

### 7. Load Data into SQL

```python
df.to_sql('df_orders', con=conn, index=False, if_exists='replace')
```

## Data Analysis
The following SQL queries are designed to perform data analysis with the objective of answering key business and analytical questions:

### Queries

1. **Top 10 Highest Revenue Generating Products :**

```sql
select top 10 product_id,sum(sale_price) as sales
from df_orders
group by product_id
order by sales desc
 ```
2. **Top 5 Highest Selling Products in Each Region:**

```sql
with cte as (
select region,product_id,sum(sale_price) as sales
from df_orders
group by region,product_id),
ranked_cte as (
select *
, row_number() over(partition by region order by sales desc) as rn
from cte) 
select * from ranked_cte
where rn<6
 ```

















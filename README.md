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

1. **Top 10 highest Revenue generating Products :**
```sql
select top 10 product_id, sum(sale_price) as sales
from df_orders
group by product_id
order by sales desc
 ```

2. **Top 5 highest selling Products in each Region :**
```sql
with cte as (
select region, product_id, sum(sale_price) as sales
from df_orders
group by region, product_id ),
ranked_cte as (
select *
, row_number() over ( partition by region order by sales desc ) as rn
from cte  ) 
select * from ranked_cte
where rn < 6
 ```

3. **Month-over-Month growth comparison for 2022 and 2023 Sales :**
```sql
with cte as (
select year(order_date) as order_year, month(order_date) as order_month,
sum(sale_price) as sales
from df_orders
group by year(order_date),month(order_date)
	)
select order_month
, sum(case when order_year=2022 then sales else 0 end) as sales_2022
, sum(case when order_year=2023 then sales else 0 end) as sales_2023
from cte 
group by order_month
order by order_month
```

4. **For each Category which Month had highest Sales :**
```sql
with cte as (
select category, format(order_date,'yyyyMM') as order_year_month
, sum(sale_price) as sales 
from df_orders
group by category, format(order_date,'yyyyMM')
),
ranked_cte as (
select *,
row_number() over(partition by category order by sales desc) as rn
from cte)
select * from ranked_cte
where rn = 1
```

5. **Which Sub-Category had highest growth by Profit in 2023 compared to 2022 :**
```sql
with cte as (
select sub_category, year(order_date) as order_year,
sum(sale_price) as sales
from df_orders
group by sub_category, year(order_date)
	),
cte2 as (
select sub_category
, sum(case when order_year=2022 then sales else 0 end) as sales_2022
, sum(case when order_year=2023 then sales else 0 end) as sales_2023
from cte 
group by sub_category
)
select top 1 *
,(sales_2023-sales_2022) as diff_in_sales
from  cte2
order by diff_in_sales desc
```

6. **How is the business performing over time (Sales and Profit Trend) :**
```sql
Select 
    year(order_date) as order_year,
    month(order_date) as order_month,
    sum(sale_price) as total_sales,
    sum(profit) as total_profit
from df_orders
group by year(order_date), month(order_date)
order by order_year, order_month
```

7. **Which Regions are driving most of the Revenue and Profit :** 
```sql
Select
    region, sum(sale_price) as total_sales, sum(profit) as total_profit
from df_orders
group by region
order by total_sales desc
```

8. **Which States are profitable and which are loss making :**
```sql
with state_profit as (
    select
        state,
        sum(sale_price) as total_sales,
        sum(profit) as total_profit
    from df_orders
    group by state
)
select *
from state_profit
where total_profit < 0
order by total_profit
```

9. **What is average profit per order by Segment :**
```sql
select
    segment,
    round(avg(profit),2) as avg_profit_per_order
from df_orders
group by segment
order by avg_profit_per_order desc
```

10. **Which Ship Modes are most profitable :**
```sql
select 
    ship_mode,
    count(order_id) as total_orders,
    sum(profit) as total_profit
from df_orders
group by ship_mode
order by total_profit desc
```

11. **What is the average order value (AOV) by Region :**
```sql
select 
    region,
    round(avg(sale_price),2) as avg_order_value
from df_orders
group by region
order by avg_order_value desc
```

12. **Monthly Profit Margin analysis :**
```sql
select 
    year(order_date) as order_year,
    month(order_date) as order_month,
    sum(profit) * 100 / sum(sale_price) as profit_margin, sum(profit) as profit, sum(sale_price) as sales
from df_orders
group by year(order_date), month(order_date)
order by order_year, order_month
```

13. **Which Segment is most sensitive to Discounts :**
```sql
select 
    segment,
    avg(discount) as avg_discount,
    sum(profit) as total_profit
from df_orders
group by segment
order by avg_discount desc
```

14. **Which Products are consistently profitable over time (not just one-time hits) :**
```sql
with monthly_profit as (
    select
        product_id,
        format(order_date, 'yyyy-MM') as order_month,
        sum(profit) as monthly_profit
    from df_orders
    group by product_id, format(order_date, 'yyyy-MM')
),
profitability_score as (
    select
        product_id,
        count(*) as total_months,
        sum(case when monthly_profit > 0 then 1 else 0 end) as profitable_months
    from monthly_profit
    group by product_id
)
select *
from profitability_score
where profitable_months * 1.0 / total_months > = 0.8
```

15. **Identify Top 3 Products per Category by Profit contribution :**
```sql
with product_profit as (
    select
        category,
        product_id,
        sum(profit) as product_profit
    from df_orders
    group by category, product_id
),
ranked_products as (
    SELECT * ,
        rank() over (partition by category order by product_profit desc) as rnk
    from product_profit
)
select *
from ranked_products
where rnk < = 3
```








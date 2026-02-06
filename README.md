# Retail Orders ETL & Analytics Project using Python and SQL

## 📋 Project Overview
This project demonstrates an end-to-end data analysis pipeline on the Retail Orders dataset using Python and SQL Server. Data is sourced via the Kaggle API, cleaned and processed using Python (Pandas), and then loaded into SQL Server for analysis. Advanced SQL concepts such as CTEs, subqueries, and window functions are used to extract meaningful business insights, showcasing a practical and scalable analytics workflow.

## Table of Contents
- [Tools and Technologies Used](#tools-and-technologies-used)
- [Getting Started](#getting-started)
- [ETL Process](#etl-process)
- [Data Analysis](#data-analysis)
- [Queries & Key Insights](#queries--key-insights)
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
The following SQL queries are designed to perform data analysis with the objective of answering key business and analytical questions :

### Queries & Key Insights

1. **Top 10 highest Revenue generating Products :**
```sql
select top 10 product_id, sum(sale_price) as sales
from df_orders
group by product_id
order by sales desc
 ```
<img width="287" height="275" alt="Output -1" src="https://github.com/user-attachments/assets/f594b22f-38fc-4013-8117-35e2d7ab3999" />

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
<img width="331" height="468" alt="Output - 2" src="https://github.com/user-attachments/assets/e179e719-636f-4489-9edc-eaeaf112fb3c" />

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
<img width="316" height="313" alt="Screenshot 2026-02-06 173348" src="https://github.com/user-attachments/assets/dc073bb2-1c95-4fa9-b27c-c57bc5451e1f" />

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
<img width="377" height="133" alt="Screenshot 2026-02-06 173712" src="https://github.com/user-attachments/assets/db306cf7-eb09-4dc1-9de6-123285d18028" />

5. **Which Sub-Category had highest growth by Profit in 2023 compared to 2022 :**
```sql
with cte as (
select sub_category, year(order_date) as order_year,
sum(profit) as profit
from df_orders
group by sub_category, year(order_date)
	),
cte2 as (
select sub_category
, sum(case when order_year=2022 then profit else 0 end) as profit_2022
, sum(case when order_year=2023 then profit else 0 end) as profit_2023
from cte 
group by sub_category
)
select top 1 *
,(profit_2023-profit_2022) as diff_in_profit
from  cte2
order by diff_in_profit desc
```
<img width="431" height="94" alt="Screenshot 2026-02-06 174209" src="https://github.com/user-attachments/assets/e8add2bd-736e-4935-96ba-195f41b38d4f" />

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
<img width="366" height="545" alt="Screenshot 2026-02-06 183325" src="https://github.com/user-attachments/assets/1aa6c35f-c77e-4235-934b-0845d8d03a30" />

7. **Which Regions are driving most of the Revenue and Profit :** 
```sql
Select
    region, sum(sale_price) as total_sales, sum(profit) as total_profit
from df_orders
group by region
order by total_sales desc
```
<img width="266" height="143" alt="Screenshot 2026-02-06 183602" src="https://github.com/user-attachments/assets/8c24eae1-25dc-48f0-9c10-5976282fb7c7" />

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
<img width="275" height="134" alt="Screenshot 2026-02-06 183835" src="https://github.com/user-attachments/assets/1c771329-0d73-42d0-994a-f09919a595e9" />

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
<img width="321" height="125" alt="Screenshot 2026-02-06 184024" src="https://github.com/user-attachments/assets/c958c353-7de1-4cc1-bcd7-d5e41f74929b" />

11. **What is the average order value (AOV) by Region :**
```sql
select 
    region,
    round(avg(sale_price),2) as avg_order_value
from df_orders
group by region
order by avg_order_value desc
```
<img width="232" height="150" alt="Screenshot 2026-02-06 184135" src="https://github.com/user-attachments/assets/dc49d0cd-dfcf-4b90-9c66-f333f57910a9" />

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
<img width="433" height="547" alt="Screenshot 2026-02-06 184401" src="https://github.com/user-attachments/assets/262f5d2d-a557-44cc-877c-b94740074721" />

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
<img width="308" height="132" alt="Screenshot 2026-02-06 184529" src="https://github.com/user-attachments/assets/1b559c09-3b5e-4df2-a00c-ab2ee63cf7f2" />

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
<img width="387" height="648" alt="Screenshot 2026-02-06 184704" src="https://github.com/user-attachments/assets/d585596d-bdce-471d-8132-7fc808cc6089" />

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
<img width="409" height="250" alt="Screenshot 2026-02-06 184755" src="https://github.com/user-attachments/assets/2d5ff99f-3f4a-4879-b07a-4591b4e0d6e3" />

## Conclusion
This project represents a complete end-to-end learning journey in data analytics, combining Python and SQL to derive meaningful insights from a retail orders dataset. From acquiring data via the Kaggle API to performing thorough data cleaning, transformation, and structured analysis in SQL Server, each phase demanded consistency, problem-solving, and attention to detail. The challenges encountered throughout the process strengthened my understanding of real-world data workflows and reinforced best practices in data handling and analysis. This work reflects not only my technical growth but also my perseverance, discipline, and genuine passion for transforming raw data into actionable business insights.


<div align="left">
<h2>Connect with me <a href="https://gifyu.com/image/Zy2f"><img src="https://github.com/milaan9/milaan9/blob/main/Handshake.gif" width="50px"></a>
</h2>
<p align="left">
 
 [<img src = "https://cdn.pixabay.com/photo/2022/01/30/13/33/github-6980894_640.png" width ="55" height ="55" align = "center" style= "postion:relative">](https://github.com/TanishkBindal)&nbsp;&nbsp;&nbsp;&nbsp;
[<img src = "https://upload.wikimedia.org/wikipedia/commons/thumb/f/f8/LinkedIn_icon_circle.svg/1024px-LinkedIn_icon_circle.svg.png" width ="55" height ="85" align = "center" style= "postion:relative">](https://www.linkedin.com/in/tanishk-bindal)
[<img src = "https://static.vecteezy.com/system/resources/previews/066/118/531/non_2x/linktree-circle-logo-icon-linktree-app-editable-transparent-background-premium-social-media-design-for-digital-download-free-png.png" width ="90" height ="280" align = "center" style= "postion:relative">](https://linktr.ee/tanishkbindal)

 
 <p align="right">(<a href="#top">Back to top</a>)</p>
</p> 













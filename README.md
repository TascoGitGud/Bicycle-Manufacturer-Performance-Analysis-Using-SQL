# Bicycle Manufacturer Performance Analysis

![SQL](https://img.shields.io/badge/Language-SQL-3776AB?style=flat-square&logo=sql&logoColor=white)
![Google BigQuery](https://img.shields.io/badge/Google_BigQuery-4285F4?style=flat-square&logo=googlebigquery&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-success?style=flat-square)

---

<p align="center">
  <img src="Images/banner.jpg" width="100%">
</p>


## 1. Overview

This project was built to practice and showcase SQL skills by solving real-world business questions using the **AdventureWorks dataset**.

### Table of Contents

- [1. Overview](#1-overview)
- [2. Dataset](#2-dataset)
- [3. Full Query Repository](#3-full-query-repository)
- [4. Project Structure](#4-project-structure)
- [5. Setup Instructions](#5-setup-instructions)

---

## 2. Dataset

The analysis is based on the **AdventureWorks database**, which represents a large bicycle manufacturing and sales company operating internationally. It contains data on products, customers, sales orders, purchasing, and inventory across many regions.

### Data Dictionary

To answer the 8 business questions in this project, **8 tables** from the `Sales`, `Production`, and `Purchasing` schemas were used. The table below lists only the columns that were actually used.

> 🔗 **Full Documentation:** For the complete Data Dictionary of the entire AdventureWorks dataset, see the [Official Data Dictionary (PDF)](https://drive.google.com/file/d/1bwwsS3cRJYOg1cvNppc1K_8dQLELN16T/view).

| Schema | Table Name | Columns Used | Purpose |
| :--- | :--- | :--- | :--- |
| **Sales** | `SalesOrderHeader` | `SalesOrderID`, `OrderDate`, `CustomerID`, `TerritoryID`, `Status`, `ModifiedDate` | Main table for order dates, territories, customer cohorts, and order status. |
| **Sales** | `SalesOrderDetail` | `SalesOrderID`, `ProductID`, `OrderQty`, `LineTotal`, `UnitPrice`, `SpecialOfferID` | Detail-level table used to calculate quantities, revenue, and pricing. |
| **Sales** | `SpecialOffer` | `SpecialOfferID`, `DiscountPct`, `Type` | Used to identify "Seasonal Discount" offers and their discount rates. |
| **Production** | `Product` | `ProductID`, `Name`, `ProductSubcategoryID` | Connects product IDs to product names and subcategories. |
| **Production** | `ProductSubcategory` | `ProductSubcategoryID`, `Name` | Groups products into subcategories for growth comparisons. |
| **Production** | `WorkOrder` | `ProductID`, `StockedQty`, `ModifiedDate` | Provides stock quantities used to track monthly inventory trends. |
| **Purchasing** | `PurchaseOrderHeader` | `PurchaseOrderID`, `Status`, `TotalDue`, `ModifiedDate` | Used to look at purchase orders with `Status = 1` (Pending). |
| **Purchasing** | `PurchaseOrderDetail` | `PurchaseOrderID` | Provides line-item detail linked to purchase orders. |

---

## 3. Full Query Repository

Below are all 8 queries with their logic and a sample of the results returned in BigQuery.

<details>
<summary><b>Query 1: Sales Volume L12M</b> (Click to expand)</summary>

*Question: Calc Quantity of items, Sales value & Order quantity by each Subcategory in L12M.*

```sql
WITH
  sales_order_with_date AS(
    SELECT
      sales_detail.SalesOrderID,
      sales_detail.ProductID,
      sales_detail.OrderQty,
      sales_detail.LineTotal,
      DATE(sales_header.OrderDate) as order_date,
      MAX(DATE(sales_header.OrderDate)) OVER() AS last_order_date,
      DATE_SUB(MAX(DATE(sales_header.OrderDate)) OVER(), INTERVAL 12 MONTH) L12M
    FROM `adventureworks2019.Sales.SalesOrderDetail` sales_detail
    INNER JOIN `adventureworks2019.Sales.SalesOrderHeader` sales_header
    ON sales_detail.SalesOrderID = sales_header.SalesOrderID
  ),
  sales_in_L12M AS(
    SELECT
      s.SalesOrderID, s.OrderQty, s.LineTotal,	
      FORMAT_DATE('%b %Y', s.order_date) period,
      p.ProductSubcategoryID,
      sub.name AS product_subcategory
    FROM sales_order_with_date s
    LEFT JOIN `adventureworks2019.Production.Product` p ON s.ProductID = p.ProductID
    LEFT JOIN `adventureworks2019.Production.ProductSubcategory` sub ON CAST(p.ProductSubcategoryID AS INT64) = sub.ProductSubcategoryID
    WHERE order_date >= L12M
  )
SELECT
  period,
  product_subcategory, 
  SUM(OrderQty) AS qty_item,
  ROUND(SUM(LineTotal), 4) as total_sales,
  COUNT(SalesOrderID) AS oder_cnt
FROM sales_in_L12M
GROUP BY period, product_subcategory
ORDER BY period DESC, product_subcategory;
```
**Actual Output:**
![Query 1 Output](Images/Query_1_Output.png)

</details>

<details>
<summary><b>Query 2: YoY Growth Rate by Category</b> (Click to expand)</summary>

*Question: Calc % YoY growth rate by SubCategory & release top 3 cat with highest grow rate. Can use metric: quantity_item. Round results to 2 decimal.*

```sql
WITH
  sales_with_date AS (
    SELECT
      sales_detail.SalesOrderID order_id,
      sales_detail.ProductID product_id,
      sales_detail.OrderQty order_qty,
      DATE(sales_header.OrderDate) as order_date,
    FROM `adventureworks2019.Sales.SalesOrderDetail` sales_detail
    INNER JOIN `adventureworks2019.Sales.SalesOrderHeader` sales_header
      ON sales_detail.SalesOrderID = sales_header.SalesOrderID
  ),
  sales_with_date_subcate_name AS (
    SELECT
      order_id, product_id, order_qty,
      EXTRACT(YEAR FROM order_date) order_year,
      p.ProductSubcategoryID product_subcate_id,
      sub.Name subcate_name
    FROM sales_with_date s
    LEFT JOIN `adventureworks2019.Production.Product` p ON s.product_id = p.ProductID
    LEFT JOIN `adventureworks2019.Production.ProductSubcategory` sub ON CAST(p.ProductSubcategoryID AS INT64) = sub.ProductSubcategoryID
  ),
  qty_sum_by_subcate AS(
    SELECT order_year, subcate_name, SUM(order_qty) qty_item
    FROM sales_with_date_subcate_name
    GROUP BY subcate_name, order_year
  ),
qty_growth AS(
  SELECT
    a.order_year, a.subcate_name, a.qty_item,
    b.order_year prev_year, b.qty_item prev_qty_item,
    ROUND((a.qty_item/b.qty_item - 1), 2) qty_diff,
    DENSE_RANK() OVER(ORDER BY ROUND((a.qty_item/b.qty_item - 1), 2) DESC) as growth_rank
  FROM qty_sum_by_subcate a
  LEFT JOIN qty_sum_by_subcate b ON a.subcate_name = b.subcate_name AND a.order_year = b.order_year + 1
 )
SELECT subcate_name Name, qty_item, prev_qty_item prv_qty, qty_diff
FROM qty_growth WHERE growth_rank <= 3 ORDER BY qty_diff DESC;
```
**Actual Output:**
![Query 2 Output](Images/Query_2_Output.png)

</details>

<details>
<summary><b>Query 3: Top Territories by Year</b> (Click to expand)</summary>

*Question: Ranking Top 3 TeritoryID with biggest Order quantity of every year. If there's TerritoryID with same quantity in a year, do not skip the rank number.*

```sql
WITH 
  territory_vs_order_count AS (
    SELECT 
      EXTRACT(YEAR FROM OrderDate) yr,
      TerritoryID,
      SUM(OrderQty) order_cnt
    FROM `adventureworks2019.Sales.SalesOrderDetail` sales_detail
    INNER JOIN `adventureworks2019.Sales.SalesOrderHeader` sales_header
      ON sales_detail.SalesOrderID = sales_header.SalesOrderID
    GROUP BY EXTRACT(YEAR FROM OrderDate), TerritoryID
  ),
  ranking_order_quantity AS(
    SELECT
      yr, TerritoryID, order_cnt,
      DENSE_RANK() OVER(PARTITION BY yr ORDER BY order_cnt DESC) rk
    FROM territory_vs_order_count
  )
SELECT * FROM ranking_order_quantity WHERE rk <= 3 ORDER BY yr DESC;
```
**Actual Output:**
![Query 3 Output](Images/Query_3_Output.png)

</details>

<details>
<summary><b>Query 4: Seasonal Discount Efficiency</b> (Click to expand)</summary>

*Question: Calc Total Discount Cost belongs to Seasonal Discount for each SubCategory.*

```sql
WITH
  combined_sales_info AS(
    SELECT
      detail.ProductID, product.ProductSubcategoryID, subcate.Name subcate_name,
      header.OrderDate order_date, detail.OrderQty order_qnt, detail.UnitPrice unit_price,
      detail.SpecialOfferID, offer.DiscountPct discount_pct, offer.Type discount_type
    FROM `adventureworks2019.Sales.SalesOrderDetail` detail
    INNER JOIN `adventureworks2019.Sales.SalesOrderHeader` header ON detail.SalesOrderID = header.SalesOrderID
    INNER JOIN `adventureworks2019.Sales.SpecialOffer` offer ON detail.SpecialOfferID = offer.SpecialOfferID
    INNER JOIN `adventureworks2019.Production.Product` product ON detail.ProductID = product.ProductID
    INNER JOIN `adventureworks2019.Production.ProductSubcategory` subcate ON CAST(product.ProductSubcategoryID AS INT64) = subcate.ProductSubcategoryID
  ),
  calculated_discount_cost AS(
    SELECT
      EXTRACT(YEAR FROM order_date) year, subcate_name,
      (discount_pct * unit_price * order_qnt) discount_cost
    FROM combined_sales_info WHERE discount_type = 'Seasonal Discount'
  )
SELECT year, subcate_name, SUM(discount_cost) total_cost
FROM calculated_discount_cost GROUP BY year, subcate_name ORDER BY year;
```
**Actual Output:**
![Query 4 Output](Images/Query_4_Output.png)

</details>

<details>
<summary><b>Query 5: Cohort Retention Rate</b> (Click to expand)</summary>

*Question: Retention rate of Customer in 2014 with status of Successfully Shipped (Cohort Analysis).*

```sql
WITH successful_order AS (
    SELECT  
      EXTRACT(MONTH FROM ModifiedDate) order_month, CustomerID customer_id
    FROM `adventureworks2019.Sales.SalesOrderHeader` 
    WHERE EXTRACT(YEAR FROM ModifiedDate) = 2014 AND Status = 5 ORDER BY customer_id, order_month
),
rank_time_order AS (
    SELECT *, ROW_NUMBER() OVER(PARTITION BY customer_id ORDER BY order_month) order_time
    FROM successful_order
),
first_order AS (
    SELECT order_month AS month_join, customer_id
    FROM rank_time_order WHERE order_time = 1
),
find_month_diff AS (
    SELECT distinct order_month, month_join, a.customer_id, (order_month - month_join) month_diff_num
    FROM successful_order a INNER JOIN first_order b ON a.customer_id = b.customer_id
    ORDER BY month_join, order_month
)
SELECT month_join, CONCAT('M-',month_diff_num) month_diff, COUNT(customer_id) customer_count
FROM find_month_diff GROUP BY month_join, CONCAT('M-',month_diff_num) ORDER BY month_join, month_diff;
```
**Actual Output:**
![Query 5 Output](Images/Query_5_Output.png)

</details>

<details>
<summary><b>Query 6: Stock Trend MoM</b> (Click to expand)</summary>

*Question: Trend of Stock level & MoM diff % by all product in 2011. If %gr rate is null then 0. Round to 1 decimal.*

```sql
WITH
  stock_info_2011 AS (
    SELECT 
      EXTRACT(YEAR FROM o.ModifiedDate) yr, EXTRACT(MONTH FROM o.ModifiedDate) mth,
      StockedQty, o.ProductID, p.Name product_name
    FROM `adventureworks2019.Production.WorkOrder` o
    INNER JOIN `adventureworks2019.Production.Product` p ON o.ProductID = p.ProductID
    WHERE EXTRACT(YEAR FROM o.ModifiedDate) = 2011  
  ),
  sum_stock_qty AS(
    SELECT product_name, mth, yr, SUM(StockedQty) stock_qty
    FROM stock_info_2011 GROUP BY product_name, mth, yr
  )
SELECT
  a.product_name, a.mth, a.yr, a.stock_qty, b.stock_qty AS stock_prv,
  IFNULL(ROUND((a.stock_qty / b.stock_qty - 1) *100.0,1), 0) diff
FROM sum_stock_qty a
LEFT JOIN sum_stock_qty b ON a.product_name = b.product_name AND a.mth = b.mth + 1 
ORDER BY product_name, a.mth DESC;
```
**Actual Output:**
![Query 6 Output](Images/Query_6_Output.png)

</details>

<details>
<summary><b>Query 7: Stock-to-Sales Ratio</b> (Click to expand)</summary>

*Question: Calc Ratio of Stock / Sales in 2011 by product name, by month. Order results by month desc, ratio desc. Round Ratio to 1 decimal.*

```sql
WITH 
  sales_2011 AS(
    SELECT 
      EXTRACT (MONTH FROM OrderDate) mth, EXTRACT (year FROM OrderDate) yr,
      detail.ProductID, p.Name, SUM(OrderQty) sales
    FROM `adventureworks2019.Sales.SalesOrderDetail` detail
    INNER JOIN `adventureworks2019.Sales.SalesOrderHeader` header ON detail.SalesOrderID = header.SalesOrderID
    INNER JOIN `adventureworks2019.Production.Product` p ON detail.ProductID = p.ProductID
    WHERE EXTRACT (year FROM OrderDate) = 2011 GROUP BY 1, 2, 3, 4
  ),
  stock_2011 AS (
    SELECT 
      EXTRACT(MONTH FROM o.ModifiedDate) mth, EXTRACT(YEAR FROM o.ModifiedDate) yr,
      o.ProductID, p.Name product_name, SUM(StockedQty) stock
    FROM `adventureworks2019.Production.WorkOrder` o
    INNER JOIN `adventureworks2019.Production.Product` p ON o.ProductID = p.ProductID
    WHERE EXTRACT(YEAR FROM o.ModifiedDate) = 2011 GROUP BY 1,2,3,4)
SELECT
  sa.mth, sa.yr, sa.ProductId, sa.Name, sales, stock, ROUND((stock/sales), 1) ratio
FROM sales_2011 sa
LEFT JOIN stock_2011 st ON sa.mth = st.mth AND sa.productID = st.productID
ORDER BY 1 DESC, 7 DESC;
```
**Actual Output:**
![Query 7 Output](Images/Query_7_Output.png)

</details>

<details>
<summary><b>Query 8: Pending Orders Breakdown</b> (Click to expand)</summary>

*Question: No of order and value at Pending status in 2014.*

```sql
SELECT
  EXTRACT(YEAR FROM header.ModifiedDate) yr, Status,
  COUNT(DISTINCT header.PurchaseOrderID) order_cnt,
  SUM(TotalDue) value
FROM `adventureworks2019.Purchasing.PurchaseOrderDetail` detail 
LEFT JOIN `adventureworks2019.Purchasing.PurchaseOrderHeader` header
  ON detail.PurchaseOrderID = header.PurchaseOrderID
WHERE EXTRACT(YEAR FROM header.ModifiedDate) = 2014 AND Status = 1
GROUP BY 1,2;
```
**Actual Output:**
![Query 8 Output](Images/Query_8_Output.png)

</details>


---

## 4. Project Structure

```text
Bicycle_Manufacturer_Performance_Analysis-main/
├── Images/                             # Screenshots of each query's result
│   ├── banner.jpg
│   ├── Query_1_Output.jpg
│   ├── Query_2_Output.jpg
│   ├── Query_3_Output.jpg
│   ├── Query_4_Output.jpg
│   ├── Query_5_Output.jpg
│   ├── Query_6_Output.jpg
│   ├── Query_7_Output.jpg
│   └── Query_8_Output.jpg
├── SQL_Queries/                        # SQL source files for each question
│   ├── Q1 Sales Volume L12M.sql
│   ├── Q2 YoY Growth Rate by Category.sql
│   ├── Q3 Top Territories by Year.sql
│   ├── Q4 Seasonal Discount Efficiency.sql
│   ├── Q5 Cohort Retention Rate.sql
│   ├── Q6 Stock Trend MoM.sql
│   ├── Q7 Stock-to-Sales Ratio.sql
│   └── Q8 Pending Orders Breakdown.sql
└── README.md
```

---

## 5. Setup Instructions

To run these queries in **Google BigQuery**:

1. **Set up a Google Cloud Platform (GCP) account:** Create one if you don't have it yet, and enable the BigQuery API.
2. **Load the dataset:** You'll need the `adventureworks2019` dataset. CSV exports of the open-source Microsoft AdventureWorks database are available online. Create a dataset named `adventureworks2019` in your BigQuery project and upload the required tables (`SalesOrderDetail`, `SalesOrderHeader`, `Product`, `WorkOrder`, etc.).
3. **Clone this repository:**
```bash
[git clone https://github.com/hdangnguyen/Bicycle_Manufacturer_Performance_Analysis.git
cd Bicycle_Manufacturer_Performance_Analysis](https://github.com/TascoGitGud/Bicycle-Manufacturer-Performance-Analysis-Using-SQL.git)
```
4. **Run the queries:** Open the BigQuery console, copy each `.sql` file's content from the `SQL_Queries/` folder, and make sure your project context matches the dataset path before running.

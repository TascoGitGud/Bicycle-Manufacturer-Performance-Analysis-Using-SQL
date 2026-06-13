# Bicycle Manufacturer Performance Analysis

![SQL](https://img.shields.io/badge/Language-SQL-3776AB?style=flat-square&logo=sql&logoColor=white)
![Google BigQuery](https://img.shields.io/badge/Google_BigQuery-4285F4?style=flat-square&logo=googlebigquery&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-success?style=flat-square)

---

<p align="center">
  <img src="Images/banner.jpg" width="100%">
</p>


## 1. Overview

This project main goal is to demonstrate advanced SQL capabilities in solving complex business problems for AdventureWorld dataset.

### Table of Contents

- [1. Overview](#1-overview)
- [2. Dataset](#2-dataset)
- [3. Full Query Repository](#3-full-query-repository)
- [4. Project Structure](#4-project-structure)
- [5. Setup Instructions](#5-setup-instructions)

---

## 2. Dataset

This project is an end-to-end data analysis performed on the **AdventureWorks database**, a comprehensive dataset simulating a large multinational manufacturing company. The business operates across multiple international regions, managing thousands of products, salespeople, and complex supply chain records.

### Data Dictionary

To execute the 8 operational queries in this project, I utilized **8 tables** across the `Sales`, `Production`, and `Purchasing` schemas. Below is a targeted data dictionary of the exact fields used in my analysis. 

> 🔗 **Full Documentation:** For the complete, un-abridged Data Dictionary of the entire AdventureWorks dataset, please refer to the [Official Data Dictionary (PDF)](https://drive.google.com/file/d/1bwwsS3cRJYOg1cvNppc1K_8dQLELN16T/view).

| Schema | Table Name | Columns Used in Queries | Business Purpose in Analysis |
| :--- | :--- | :--- | :--- |
| **Sales** | `SalesOrderHeader` | `SalesOrderID`, `OrderDate`, `CustomerID`, `TerritoryID`, `Status`, `ModifiedDate` | Base table for tracking cohort timelines, territory performance, and successful conversions. |
| **Sales** | `SalesOrderDetail` | `SalesOrderID`, `ProductID`, `OrderQty`, `LineTotal`, `UnitPrice`, `SpecialOfferID` | Fact table for aggregating total demand, volume, and revenue. |
| **Sales** | `SpecialOffer` | `SpecialOfferID`, `DiscountPct`, `Type` | Sourced the "Seasonal Discount" type and percentages for cost-efficiency tracking. |
| **Production** | `Product` | `ProductID`, `Name`, `ProductSubcategoryID` | Dimension table linking SKU IDs to human-readable names and category clusters. |
| **Production** | `ProductSubcategory` | `ProductSubcategoryID`, `Name` | Used to group specific bicycle models into high-level subcategories for YoY growth tracking. |
| **Production** | `WorkOrder` | `ProductID`, `StockedQty`, `ModifiedDate` | Core table specifying historical stocked quantities to measure month-over-month supply trends. |
| **Purchasing** | `PurchaseOrderHeader` | `PurchaseOrderID`, `Status`, `TotalDue`, `ModifiedDate` | Evaluated supplier backend performance by isolating `Status = 1` (Pending) orders. |
| **Purchasing** | `PurchaseOrderDetail` | `PurchaseOrderID` | Joined context for purchase order line items. |

---

## 3. Full Query Repository

Below is the execution of all 8 operational queries. They are presented here with their logic and a sample of their output results so you can explore the insights directly.

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
![Query 1 Output](documents/q1.png)

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
![Query 2 Output](documents/q2.png)

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
![Query 3 Output](documents/q3.png)

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
![Query 4 Output](documents/q4.png)

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
![Query 5 Output](documents/q5.png)

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
![Query 6 Output](documents/q6.png)

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
![Query 7 Output](documents/q7.png)

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
![Query 8 Output](documents/q8.png)

</details>


---

## 4. Project Structure

```text
Bicycle_Manufacturer_Performance_Analysis/
├── documents/                         # Contains query result output images
│   ├── q1.png
│   ├── ...
│   └── q8.png
├── query/                             
│   ├── q1_sales_performance_l12m.sql
│   ├── q2_yoy_growth_top_categories.sql
│   ├── q3_top_territories.sql
│   ├── q4_seasonal_discount_cost.sql
│   ├── q5_retention_rate_cohort.sql
│   ├── q6_stock_trend_mom.sql
│   ├── q7_stock_to_sales_ratio.sql
│   └── q8_pending_orders_2014.sql
└── README.md                          
```

---

## 5. Setup Instructions

To execute these queries on **Google BigQuery**, follow these steps:

1. **Set up Google Cloud Platform (GCP):** If you don't have one, create a GCP account and enable the BigQuery API.
2. **Import the Dataset:** You need the `adventureworks2019` dataset. You can find CSV exports of the open-source Microsoft AdventureWorks database online. Create a new dataset named `adventureworks2019` in your BigQuery project and upload the required tables (`SalesOrderDetail`, `SalesOrderHeader`, `Product`, `WorkOrder`, etc.).
3. **Clone this repository:**
```bash
git clone https://github.com/hdangnguyen/Bicycle_Manufacturer_Performance_Analysis.git
cd Bicycle_Manufacturer_Performance_Analysis
```
4. **Execute Queries:** Open the BigQuery console. Copy the query syntax from the `.sql` files within the `query/` directory. Ensure your BigQuery project context matches the query namespaces before running them. No additional Python dependencies are required.

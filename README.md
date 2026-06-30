# 🚲 SQL | Bicycle Manufacturer Performance Analysis

![SQL](https://img.shields.io/badge/Language-SQL-3776AB?style=flat-square&logo=sql&logoColor=white)
![Google BigQuery](https://img.shields.io/badge/Google_BigQuery-4285F4?style=flat-square&logo=googlebigquery&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-success?style=flat-square)

<p align="center">
  <img src="Images/main_banner.jpg" width="100%">
</p>

_Analyze sales, inventory, and purchasing data to answer 8 business questions and turn raw data into clear insights._

- 🎯 **Business Question:** Which products, territories, and time periods drive the most sales - and where are the risks?
- 🏬 **Domain:** Manufacturing & Retail
- 🛠️ **Tools:** SQL (Google BigQuery)

👤 Author: Bạch Minh Nam

---

### 📑 Table of Contents

- [📌 Overview](#-overview)
- [📂 Dataset](#-dataset)
- [🔎 Query Repository](#-query-repository)
- [🎯 Key Findings](#-key-findings)

---

## 📌 Overview

**🎯 Objective:**

- This project uses SQL (Google BigQuery) to analyze sales, production, and purchasing data from the **AdventureWorks dataset**
- It answers 8 specific business questions covering **Sales Performance, Customer Retention, and Inventory Optimization**
- The goal is to turn raw transactional and operational data into clear, actionable insights for the business

**❓ Main business question:**

This project uses SQL to analyze sales, inventory, and purchasing data from AdventureWorks to:
- Identify which product categories, territories, and time periods drive the most sales
- Evaluate how discounts, customer retention, and stock levels affect overall business performance

**👤 Who is this project for?**

- **Data analysts & business analysts** who want a reference for writing analytical SQL (CTEs, window functions, cohort analysis)
- **Decision-makers & stakeholders** who need quick insights into sales trends, inventory health, and supplier performance

---

## 📂 Dataset

The analysis is based on the **AdventureWorks database**, which represents a large bicycle manufacturing and sales company operating internationally. It contains data on products, customers, sales orders, purchasing, and inventory across many regions.

### Data Dictionary

To answer the 8 business questions in this project, **8 tables** from the `Sales`, `Production`, and `Purchasing` schemas were used. The table below lists only the columns that were actually used.

| Schema | Table Name | Columns Used | Used In | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **Sales** | `SalesOrderHeader` | `SalesOrderID`, `OrderDate`, `CustomerID`, `TerritoryID`, `Status`, `ModifiedDate` | Q1, Q2, Q3, Q4, Q5 | Provides order dates, territory IDs, customer IDs, and order status for sales-side queries. |
| **Sales** | `SalesOrderDetail` | `SalesOrderID`, `ProductID`, `OrderQty`, `LineTotal`, `UnitPrice`, `SpecialOfferID` | Q1, Q2, Q3, Q4, Q7 | Line-item table holding order quantities, revenue, and prices used to calculate sales volume and totals. |
| **Sales** | `SpecialOffer` | `SpecialOfferID`, `DiscountPct`, `Type` | Q4 | Identifies "Seasonal Discount" offers and their discount percentages to calculate discount cost. |
| **Production** | `Product` | `ProductID`, `Name`, `ProductSubcategoryID` | Q1, Q2, Q4, Q6, Q7 | Maps product IDs to product names and subcategory IDs. |
| **Production** | `ProductSubcategory` | `ProductSubcategoryID`, `Name` | Q1, Q2, Q4 | Groups products into subcategories for sales volume and YoY growth comparisons. |
| **Production** | `WorkOrder` | `ProductID`, `StockedQty`, `ModifiedDate` | Q6, Q7 | Supplies stocked quantities by month, used for stock trend and stock-to-sales ratio. |
| **Purchasing** | `PurchaseOrderHeader` | `PurchaseOrderID`, `Status`, `TotalDue`, `ModifiedDate` | Q8 | Provides purchase order status and total value to find Pending (`Status = 1`) orders in 2014. |
| **Purchasing** | `PurchaseOrderDetail` | `PurchaseOrderID` | Q8 | Joined to the purchase order header to count distinct pending purchase orders. |

> 🔗 **Full Documentation:** For the complete Data Dictionary of the entire AdventureWorks dataset, see the [AdventureWorks Data Dictionary (PDF)](https://drive.google.com/file/d/1bwwsS3cRJYOg1cvNppc1K_8dQLELN16T/view).

---

## 🔎 Query Repository

### Query 1: Sales Volume L12M

*Question: Calc Quantity of items, Sales value & Order quantity by each Subcategory in L12M.*

> _Tracking the last 12 months of sales by subcategory helps the business spot which product lines are growing or declining in real time - so inventory and marketing budgets can be adjusted before it's too late._

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
ORDER BY total_sales DESC, product_subcategory;
```

**📊 Actual Output:**
![Query 1 Output](Images/Query_01_Output.png)

**💡 Observations:**

Road Bikes dominates with $2.1M revenue (Mar 2014, 2,371 units). Three categories (Road, Mountain, Touring) drive ~80% of revenue. Maintain inventory strength and prioritize best-selling categories.

---

### Query 2: YoY Growth Rate by Category

*Question: Calc % YoY growth rate by SubCategory & release top 3 cat with highest grow rate. Can use metric: quantity_item. Round results to 2 decimal.*

> _Identifying the top 3 fastest-growing subcategories gives leadership a clear signal of where demand is heading - useful for production planning and deciding where to invest next._

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

**📊 Actual Output:**
![Query 2 Output](Images/Query_2_Output.png)

**💡 Observations:**

Mountain Frames: +510%, Socks: +421%, Road Frames: +389% YoY growth. These three categories show explosive demand. Increase production capacity and marketing investment immediately.

---

### Query 3: Top Territories by Year

*Question: Ranking Top 3 TeritoryID with biggest Order quantity of every year. If there's TerritoryID with same quantity in a year, do not skip the rank number.*

> _Knowing which territories consistently drive the most orders helps the sales team prioritize regional resources and flag underperforming areas that may need support._

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

**📊 Actual Output:**
![Query 3 Output](Images/Query_3_Output.png)

**💡 Observations:**

Territory 4 dominates (3,238→11,632 orders, 2011-2014), 4x stronger than others. Territories 1 & 6 consistently rank 2-3. Investigate Territory 4 success factors and replicate across regions.

---

### Query 4: Seasonal Discount Efficiency

*Question: Calc Total Discount Cost belongs to Seasonal Discount for each SubCategory.*

> _Calculating the total cost of seasonal discounts per subcategory lets the finance team evaluate whether the promotions are worth the margin loss - and which categories are eating the most discount budget._

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

**📊 Actual Output:**
![Query 4 Output](Images/Query_4_Output.png)

**💡 Observations:**

Only Helmets receive seasonal discounts ($828→$1,606, 2012-2013). Profitability impact unclear. Conduct ROI analysis before expanding discount strategy.

---

### Query 5: Cohort Retention Rate

*Question: Retention rate of Customer in 2014 with status of Successfully Shipped (Cohort Analysis).*

> _Cohort retention shows exactly when customers stop coming back after their first purchase - giving the CRM team a window to step in with re-engagement campaigns before churn becomes permanent._

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

**📊 Actual Output:**
![Query 5 Output](Images/Query_5_Output.png)

**💡 Observations:**

96% of 2014 customers never return (2,076→78 by month 2). Seasonal uptick at month 3 suggests re-engagement opportunity. Launch email campaigns timed to seasonal patterns.

---

### Query 6: Stock Trend MoM

*Question: Trend of Stock level & MoM diff % by all product in 2011. If %gr rate is null then 0. Round to 1 decimal.*

> _Month-over-month stock changes reveal whether inventory is building up or running low for each product - helping the warehouse team avoid both overstock and stockout situations._

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
ORDER BY a.stock_qty DESC;
```

**📊 Actual Output:**
![Query 6 Output](Images/Query_06_Output.png)

**💡 Observations:**

Stock surges (HL Mountain Frame: 5→59 units) indicate reactive restocking. Implement demand forecasting to avoid both stock-outs and overstock situations.

---

### Query 7: Stock-to-Sales Ratio

*Question: Calc Ratio of Stock / Sales in 2011 by product name, by month. Order results by month desc, ratio desc. Round Ratio to 1 decimal.*

> _A high stock-to-sales ratio means the company is holding more inventory than it's selling - tying up cash. This query flags which products need faster turnover or reduced production._

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

**📊 Actual Output:**
![Query 7 Output](Images/Query_7_Output.png)

**💡 Observations:**

Stock-to-sales ratio >20x for slow-moving frames (HL Mountain Frame Black: 27 in stock, 1 sold). Free up $M+ in working capital by reducing overstock and improving inventory turnover.

---

### Query 8: Pending Orders Breakdown

*Question: No of order and value at Pending status in 2014.*

> _Pending purchase orders represent committed but undelivered spend - tracking their total value helps the procurement team manage cash flow and follow up with suppliers before delays impact production._

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

**📊 Actual Output:**
![Query 8 Output](Images/Query_8_Output.png)

**💡 Observations:**

$9.27M in 224 pending purchase orders (2014). Establish supplier SLA monitoring and follow-up protocols to prevent production delays and cash flow bottlenecks.

---

## 🎯 Key Findings

### 1. Commercial & Growth Strategy

*   **High Revenue Concentration:** Just 3 categories (Road, Mountain, and Touring Bikes) generate **~80% of total revenue**, with Road Bikes dominating at $2.1M in March 2014 alone.
    *   *Action:* Safeguard the supply chain for these core drivers while selectively diversifying the product portfolio to reduce dependency risks.
*   **Explosive Demand Segments:** Mountain Frames (+510% YoY), Socks (+421% YoY), and Road Frames (+389% YoY) are showing massive momentum.
    *   *Action:* Immediately scale up production capacity and allocate aggressive marketing budgets to capture this market surge.
*   **Regional Powerhouse:** Territory 4 drives **4x more order volume** than any other region from 2011 to 2014.
    *   *Action:* Audit Territory 4’s local strategies (pricing, distribution, or sales execution) and replicate that playbook across underperforming territories.

### 2. Inventory & Supply Chain Optimization

*   **Capital Tied Up in Excess Stock:** The Stock-to-Sales ratio is critically high at **>20x** for slow-moving items (e.g., *HL Mountain Frame Black* holding 27 units in stock with only 1 sold).
    *   *Action:* Implement dynamic demand forecasting instead of reactive restocking to liquidate slow-moving inventory and free up **$M+ in trapped working capital**.
*   **Procurement & Pipeline Risks:** There is a staggering **$9.27M locked in 224 pending purchase orders** in 2014.
    *   *Action:* Establish strict supplier SLA monitoring and automated follow-up protocols to prevent severe production bottlenecks and secure the cash flow pipeline.

### 3. Customer Retention & Promotion Efficiency

*   **Retention Alarm:** A massive **96% of 2014 customers never returned** after their first purchase (crashing from 2,076 to just 78 active customers by Month 2).
    *   *Action:* Shift budget away from low-impact seasonal discounts (currently only utilized by Helmets with unproven ROI) and invest heavily in a dedicated CRM retention system.
*   **The "Month 3" Opportunity:** The cohort analysis flags a slight seasonal uptick in customer re-engagement exactly at **Month 3 (M-3)**.
    *   *Action:* Launch automated email marketing and personalized loyalty rewards timed precisely at the 90-day mark to intercept churning users when they are most likely to re-engage.
 
---

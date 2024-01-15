# SQL-Bicycle-Manufacturer
## Dataset: adventureworks2019
## Data Dictionary: https://drive.google.com/file/d/1bwwsS3cRJYOg1cvNppc1K_8dQLELN16T/view?usp=share_link
## The goal of project:

## Question:
-Q1: Calc Quantity of items, Sales value & Order quantity by each Subcategory in L12M
```
SELECT 
  FORMAT_TIMESTAMP("%b %Y ", sale.ModifiedDate) AS Period,
  sub.Name AS Name,
  SUM(sale.OrderQty) AS Qty_item,
  SUM(sale.LineTotal) AS total_sale,
  COUNT(DISTINCT sale.SalesOrderID) AS order_cnt
FROM 'adventureworks2019.Sales.SalesOrderDetail' AS sale
LEFT JOIN 'adventureworks2019.Production.Product' AS product
  ON sale.ProductID = product.ProductID
LEFT JOIN 'adventureworks2019.Production.ProductSubcategory' AS sub
  ON CAST(product.ProductSubcategoryID AS INT) = sub.ProductSubcategoryID
WHERE DATE(sale.ModifiedDate) BETWEEN (DATE_SUB('2014-06-30', INTERVAL 12 month)) AND '2014-06-30'
GROUP BY 1 , 2 
ORDER BY Period DESC, Name;
```
-Q2: Calc % YoY growth rate by SubCategory & release top 3 cat with highest grow rate. Can use metric: quantity_item. Round results to 2 decimal
```
WITH sale_info AS (   
  SELECT 
    Name, 
    qty_item, 
    LAG(qty_item) OVER(PARTITION BY NAME ORDER BY YR) AS prv_qty 
  FROM 
    (SELECT 
      sub.Name,
      SUM(OrderQty) AS qty_item,
      EXTRACT(YEAR FROM SALE.ModifiedDate) AS YR
    FROM 
      'adventureworks2019.Sales.SalesOrderDetail' AS SALE
    LEFT JOIN 
      'adventureworks2019.Production.Product' AS product
    ON sale.ProductID = product.ProductID
    LEFT JOIN 
      'adventureworks2019.Production.ProductSubcategory' AS sub
    ON CAST(product.ProductSubcategoryID AS INT) = sub.ProductSubcategoryID
    GROUP BY 1,3)
)
SELECT * , 
  ROUND((qty_item - prv_qty) / prv_qty , 2) AS qty_diff
FROM cte
ORDER BY qty_diff DESC
LIMIT 3;
```
-Q3: Ranking Top 3 TeritoryID with biggest Order quantity of every year. If there's TerritoryID with same quantity in a year, do not skip the rank number
```
WITH ranking AS(
  SELECT 
    YR,
    TerritoryID,
    Order_quantity,
    dense_RANK()OVER(partition by YR ORDER BY  Order_quantity desc) as rk
    --RANK()OVER(partition by YR ORDER BY  Order_quantity desc) as rk
    --> mình nên thay bằng dense_rank để luôn đảm bảo k bị skip rank
  FROM 
    (SELECT 
      TerritoryID,
      SUM(OrderQty) AS Order_quantity,
      EXTRACT(YEAR FROM SALE_DETAIL.ModifiedDate) AS YR,
    FROM 'adventureworks2019.Sales.SalesOrderDetail' AS SALE_DETAIL 
    LEFT JOIN 'adventureworks2019.Sales.SalesOrderHeader' AS SALE_HEADER
      ON Sale_Detail.SalesOrderID = Sale_Header.SalesOrderID
    GROUP BY 1, 3
))
SELECT *
FROM ranking
WHERE rk <= 3;
```
-Q4: Calc Total Discount Cost belongs to Seasonal Discount for each SubCategory
```
SELECT 
  EXTRACT( YEAR FROM SOD.ModifiedDate) AS YR,
  PPS.Name,
  SUM(OrderQty * UnitPrice * DiscountPct) AS total_discount
FROM 'adventureworks2019.Sales.SalesOrderDetail' AS SOD
LEFT JOIN 'adventureworks2019.Sales.SpecialOffer' AS SO USING(SpecialOfferID) 
LEFT JOIN 'adventureworks2019.Production.Product' AS PP USING(ProductID)
LEFT JOIN 'adventureworks2019.Production.ProductSubcategory' AS PPS
  ON CAST(PPS.ProductSubcategoryID AS STRING) = PP.ProductSubcategoryID
WHERE SO.Type  = 'Seasonal Discount'     
GROUP BY 1,2
ORDER BY 1;
```
-Q5: Retention rate of Customer in 2014 with status of Successfully Shipped (Cohort Analysis)
```
with info as (
  select 
    extract(year from ModifiedDate) as yr,
    extract(month from ModifiedDate) as mth,
    CustomerID,
    count(distinct SalesOrderID) as sale_cnt
  from 'adventureworks2019.Sales.SalesOrderHeader'
  where Status = 5 and extract(year from ModifiedDate) = 2014
  group by 1,2,3
)
,row_num as (
  select *
    ,row_number() over(partition by CustomerID order by mth) as num
  from info
)
,first_order as (
  select 
    distinct mth as month_join,
    CustomerID
  from row_num
  where num = 1)
select 
  first_order.month_join,
  concat('M-',info.mth - first_order.month_join) as month_diff,
  count(CustomerID) as customer_cnt
from info
left join first_order
  using(CustomerID)
group by 1,2
order by 1,2
;
```
-Q6: Trend of Stock level & MoM diff % by all product in 2011. If %gr rate is null then 0. Round to 1 decimal
```
WITH stock_level_trend AS (
  SELECT
    PP.Name,
    EXTRACT(MONTH FROM PWO.ModifiedDate) AS Month,
    SUM(PWO.StockedQty) AS StockQty
  FROM adventureworks2019.Production.WorkOrder AS PWO
  LEFT JOIN adventureworks2019.Production.Product AS PP
    ON PWO.ProductID = PP.ProductID
  WHERE EXTRACT(YEAR FROM PWO.ModifiedDate) = 2011
  GROUP BY 1, 2
  ORDER BY 1, 2
)
SELECT
  Name,
  Month,
  extract(year from a.ModifiedDate) as yr ,
  StockQty,
  COALESCE(ROUND((StockQty - LAG(StockQty, 1) OVER (PARTITION BY Name ORDER BY Month)) / LAG(StockQty, 1) OVER (PARTITION BY Name ORDER BY Month) * 100, 1), 0) AS Diff
FROM stock_level_trend
ORDER BY 1, 2 DESC;
```
-Q7: Calc Ratio of Stock / Sales in 2011 by product name, by month
```
with sales_info as (
  select
    extract(month from SOD.ModifiedDate) as mth,
    ProductID, name,
    sum(OrderQty) as sales,
  from adventureworks2019.Sales.SalesOrderDetail AS SOD
  LEFT join adventureworks2019.Production.Product as PP
    using(ProductID)
  where FORMAT_TIMESTAMP("%Y", a.ModifiedDate) = '2011'
  group by 1 ,2, 3
  order by 1 desc
)
,stock_info as (
  select 
    extract(month from ModifiedDate) as mth,
    ProductID,
    sum(StockedQty) as stock
  from adventureworks2019.Production.WorkOrder
  where FORMAT_TIMESTAMP("%Y", a.ModifiedDate) = '2011'
  group by 1 ,2
  order by 1 desc
)
select 
  sales_info.mth,
  sales_info.name,
  stock,sales,
  round(stock/sales, 1) as ratio
from sales_info
inner join stock_info
  on sales_info.ProductID = stock_info.ProductID
  and sales_info.mth = stock_info.mth
order by 1 desc, ratio desc
;
```
-Q8: No of order and value at Pending status in 2014
```
select 
    extract (year from ModifiedDate) as yr
    , Status
    , count(distinct PurchaseOrderID) as order_Cnt 
    , sum(TotalDue) as value
from `adventureworks2019.Purchasing.PurchaseOrderHeader`
where Status = 1
and extract(year from ModifiedDate) = 2014
group by 1,2
;
```

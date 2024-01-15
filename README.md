# SQL-Bicycle-Manufacturer
# Dataset: adventureworks2019
# Data Dictionary: https://drive.google.com/file/d/1bwwsS3cRJYOg1cvNppc1K_8dQLELN16T/view?usp=share_link
# The goal of project:

# Question:
--Q1
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

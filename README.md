# MySQL-Business-Problems-Practice

## Objective
The objective of the Practice project is to solve business problems by using MySQL. It attempts to solve  problems using Basic to advanced Mysql functions and answer business queries to make data driven decisions and also hone SQL programming skills.
Functions that are used in this practice project: Joins, Aggregate, Case, Subquery, Window functions such as row_number, Rank, Dense_rank, LAG  etc.

Below is a brief description of the dataset: 
This dataset simulates an e-commerce business selling products like electronics, books, clothing, toys, and furniture. Customers place orders online using various payment methods, and the company tracks stock, customer demographics, and delivery timelines.
1. customers_cleaned.csv – Contains customer details
![Image](https://github.com/user-attachments/assets/109d140f-f64b-474a-acab-b2a9a97ce73e)

2. orders.csv – Contains order records linked to customers and products

![Image](https://github.com/user-attachments/assets/df6f3ef4-8c56-444c-b5b5-62946292983d)

3. products_cleaned.csv – Contains product information

![Image](https://github.com/user-attachments/assets/0f2b8582-5707-4be1-8325-e21f06d2a7d8)

   ## Problems Solving:

## Find the top Customers by Revenue.

```mysql
SELECT name, ROUND(
        SUM(p.price * orders.quantity), 2
    ) as revenue, ROW_NUMBER() OVER (
        ORDER BY ROUND(
                SUM(p.price * orders.quantity), 2
            ) DESC
    ) AS ranks
FROM
    products_cleaned p
    JOIN orders ON p.product_id = orders.product_id
    JOIN customers_cleaned c ON c.customer_id = orders.customer_id
GROUP BY
    1
ORDER BY 2 DESC
LIMIT 3;
```

##  How many orders were delivered late (more than 7 days), and what’s the average delay per product category?
```mysql
SELECT p.category, COUNT(*) as late_orders, ROUND(
        AVG(
            DATEDIFF(
                CAST(o.delivery_date as DATE), CAST(o.order_date as DATE)
            )
        ), 2
    ) AS avg_delay
FROM orders o
    JOIN products_cleaned p ON p.product_id = o.product_id
WHERE
    DATEDIFF(
        CAST(o.delivery_date as DATE),
        CAST(o.order_date as DATE)
    ) > 7
GROUP BY
    1;
```

## Identify the repeat customers and the amount they purchased.

```mysql
SELECT
    name,
    COUNT(o.customer_id) AS purchase_count,
    ROUND(SUM(p.price * o.quantity), 2) AS total_purchase
FROM
    customers_cleaned c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN products_cleaned p ON p.product_id = o.product_id
GROUP BY
    1
HAVING
    COUNT(o.customer_id) > 1
ORDER BY 2 DESC;
```

## Identify the top 10 best seller products

```mysql
SELECT
    product_name,
    ROUND(SUM(p.price * o.quantity), 2) AS revenue,
    RANK() OVER (
        ORDER BY SUM(p.price * o.quantity) desc
    ) as rank_revenue
FROM products_cleaned p
    JOIN orders o ON p.product_id = o.product_id
GROUP BY
    1
ORDER BY 2 DESC
LIMIT 10;
```

## Find the number of cancelled orders and revenue lost for each month

```mysql
SELECT
    DATE_FORMAT(
        CAST(o.order_date AS DATE),
        '%Y-%m'
    ) AS month,
    ROUND(SUM(p.price * o.quantity), 2) as revenue,
    COUNT(DISTINCT (order_id)) AS cancelled_orders
FROM orders o
    JOIN products_cleaned p ON o.product_id = p.product_id
WHERE
    status = 'Cancelled'
GROUP BY
    1;
```

##   Find the top 3  brands for each product category

```mysql
WITH
    cte AS (
        SELECT p.category, p.brand, ROUND(SUM(p.price * o.quantity), 2) as revenue, DENSE_RANK() OVER (
                PARTITION BY
                    p.category
                ORDER BY SUM(p.price * o.quantity) DESC
            ) as ranked
        FROM products_cleaned p
            join orders o ON p.product_id = o.product_id
        GROUP BY
            1, 2
    )
SELECT *
from cte
WHERE
    ranked <= 3;
```

## Find the products which generated more than the average revenue from total sales

```mysql
SELECT product_name, ROUND(SUM(p.price * o.quantity), 2) as revenue
FROM products_cleaned p
    JOIN orders o ON p.product_id = o.product_id
GROUP BY 1
HAVING
    SUM(p.price * o.quantity) > (
        SELECT AVG(p.price * o.quantity) as avg_rev
        FROM products_cleaned p
            JOIN orders o ON p.product_id = o.product_id
    )
ORDER BY 2 DESC;
```

## The company wants to know which products are in high demand and needs to be restocked quickly. Categorize the products based on their current stock and revenue.

```mysql
WITH
    cte AS (
        SELECT
            product_name,
            p.stock_quantity,
            ROUND(SUM(p.price * o.quantity), 2) as revenue,
            (
                SELECT AVG(stock_quantity) AS avg_stock
                FROM products_cleaned
            ) as avg_stock
        FROM products_cleaned p
            JOIN orders o ON p.product_id = o.product_id
        GROUP BY 1, 2
        ORDER BY 3 desc
    )
SELECT
    product_name,
    revenue,
    CASE
        WHEN revenue > 75000
        AND stock_quantity < avg_stock THEN 'High'
        WHEN revenue > 60000
        AND stock_quantity < avg_stock THEN 'Moderate'
        WHEN revenue > 40000
        AND stock_quantity < avg_stock THEN 'Low'
        ELSE 'Very Low'
    END restock_priority
FROM cte;
```

##  Analyze month over month performance by finding the percentage change in revenue between the current & previous month

```mysql
SELECT 
    months, 
    current_revenue, 
    CONCAT(ROUND((current_revenue - previous_month) / previous_month * 100, 2),'%') AS percent_change
FROM (
    SELECT  
        MONTH(order_date) AS months,
        ROUND(SUM(p.price * o.quantity), 2) AS current_revenue,
        ROUND(
            LAG(SUM(p.price * o.quantity)) OVER (ORDER BY MONTH(order_date)), 
            2
        ) AS previous_month
    FROM 
        orders o
    JOIN 
        products_cleaned p ON o.product_id = p.product_id
    WHERE 
        status = 'Completed'
    GROUP BY 
        MONTH(order_date)
    ORDER BY 
        MONTH(order_date)
) subquery;
```





















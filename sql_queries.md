# SQL Queries Documentation

## Ad-Hoc Request 1: Provide the list of markets in which customer "Atliq Exclusive" operates its business in the APAC region

```sql
SELECT DISTINCT market
FROM gdb023.dim_customer
WHERE customer = 'Atliq Exclusive' AND region = 'APAC';
```
## Ad-Hoc Request 2: What is the percentage of unique product increase in 2021 vs. 2020?

```sql
WITH tot_products AS (
  SELECT 
    Count(DISTINCT product_code) AS total_products, 
    fiscal_year AS year 
  FROM 
    fact_sales_monthly 
  GROUP BY 
    fiscal_year
) 
SELECT 
  a.total_products AS unique_products_2020, 
  b.total_products AS unique_products_2021, 
  (b.total_products - a.total_products)
 AS new_products_introduced, 
round (
    (
      b.total_products - a.total_products
    ) / a.total_products * 100, 
    2
  ) AS pct_change 
  FROM 
    tot_products AS a 
    LEFT JOIN tot_products AS b ON a.year + 1 = b.year 
  limit 
    1 ;

```
## Ad-Hoc Request 3: Provide a report with all the unique product counts for each segment and sort them in descending order of product counts.

```sql
SELECT segment, COUNT(DISTINCT product_code) AS product_count
FROM dim_product
GROUP BY segment
ORDER BY product_count DESC;
```
## Ad-Hoc Request 4: Which segment had the most increase in unique products in 2021 vs 2020?

```sql
WITH temp_table AS (
    SELECT 
        p.segment,
        s.fiscal_year,
        COUNT(DISTINCT s.Product_code) as product_count
    FROM 
        fact_sales_monthly s
        JOIN dim_product p ON s.product_code = p.product_code
    GROUP BY 
        p.segment,
        s.fiscal_year
)
SELECT 
    up_2020.segment,
    up_2020.product_count as product_count_2020,
    up_2021.product_count as product_count_2021,
    up_2021.product_count - up_2020.product_count as difference
FROM 
    temp_table as up_2020
JOIN 
    temp_table as up_2021
ON 
    up_2020.segment = up_2021.segment
    AND up_2020.fiscal_year = 2020 
    AND up_2021.fiscal_year = 2021
ORDER BY 
    difference DESC;
```
## Ad-Hoc Request 5: Get the products that have the highest and lowest manufacturing costs.
```sql
SELECT 
    'Highest' AS cost_type,
    fmc_high.product_code,
    fmc_high.product,
    fmc_high.manufacturing_cost AS highest_manufacturing_cost
FROM 
    (SELECT 
         fmc.product_code, 
         dp.product,
         fmc.manufacturing_cost
     FROM 
         `gdb023`.`fact_manufacturing_cost` fmc
     JOIN 
         `gdb023`.`dim_product` dp ON fmc.product_code = dp.product_code
     ORDER BY 
         fmc.manufacturing_cost DESC
     LIMIT 1) AS fmc_high

UNION ALL

SELECT 
    'Lowest' AS cost_type,
    fmc_low.product_code,
    fmc_low.product,
    fmc_low.manufacturing_cost AS lowest_manufacturing_cost
FROM 
    (SELECT 
         fmc.product_code, 
         dp.product,
         fmc.manufacturing_cost
     FROM 
         `gdb023`.`fact_manufacturing_cost` fmc
     JOIN 
         `gdb023`.`dim_product` dp ON fmc.product_code = dp.product_code
     ORDER BY 
         fmc.manufacturing_cost ASC
     LIMIT 1) AS fmc_low;

```
## Ad-Hoc Request 6: Generate a report which contains the top 5 customers who received an average high pre_invoice_discount_pct for the fiscal year 2021 and in the Indian market. 

```sql
SELECT
    fm.customer_code,
    dc.customer,
    AVG(fm.pre_invoice_discount_pct) AS average_discount_percentage
FROM
    `gdb023`.`fact_pre_invoice_deductions` fm
JOIN
    `gdb023`.`dim_customer` dc ON fm.customer_code = dc.customer_code
WHERE
    fm.fiscal_year = 2021
    AND dc.market = 'India'
GROUP BY
    fm.customer_code, dc.customer
ORDER BY
    average_discount_percentage DESC
LIMIT 5;
```
## Ad-Hoc Request 7: Get the complete report of the Gross sales amount for the customer “Atliq Exclusive” for each month. This analysis helps to get an idea of low and high-performing months and take strategic decisions.

```sql
SELECT YEAR(date) AS YEAR,
       MONTH(date) AS MONTH,
       sum(sold_quantity * gross_price) AS gross_sales_amount
FROM fact_sales_monthly AS fs
INNER JOIN fact_gross_price AS fp ON fs.product_code = fp.product_code
AND fs.fiscal_year = fp.fiscal_year
INNER JOIN dim_customer AS dc ON fs.customer_code = dc.customer_code
WHERE customer = "Atliq Exclusive"
GROUP BY MONTH,
         YEAR(date)
ORDER BY YEAR,
         MONTH ;
```
## Ad-Hoc Request 8: In which quarter of 2020, got the maximum total_sold_quantity?

```sql
WITH temp_table AS
  (SELECT date,month(date_add(date,interval 4 MONTH)) AS period,
               fiscal_year,
               sold_quantity
   FROM fact_sales_monthly)
SELECT CASE
           WHEN period/3 <= 1 THEN "Q1"
           WHEN period/3 <= 2
                AND period/3 > 1 THEN "Q2"
           WHEN period/3 <=3
                AND period/3 > 2 THEN "Q3"
           WHEN period/3 <=4
                AND period/3 > 3 THEN "Q4"
       END QUARTER,
           round(sum(sold_quantity)/1000000, 2) AS total_sold_quanity
FROM temp_table
```
## Ad-Hoc Request 9: Which channel helped to bring more gross sales in the fiscal year 2021 and the percentage of contribution?

```sql
SELECT 
    dc.channel,
    SUM(fgm.sold_quantity * fgp.gross_price) AS gross_sales_mln,
    SUM(fgm.sold_quantity * fgp.gross_price) / total.total_sales AS percentage
FROM 
    gdb023.fact_sales_monthly fgm
JOIN 
    gdb023.dim_customer dc ON fgm.customer_code = dc.customer_code
JOIN 
    gdb023.fact_gross_price fgp ON fgm.product_code = fgp.product_code
JOIN 
    (SELECT 
         SUM(fgm.sold_quantity * fgp.gross_price) AS total_sales
     FROM 
         gdb023.fact_sales_monthly fgm
     JOIN 
         gdb023.fact_gross_price fgp ON fgm.product_code = fgp.product_code
     WHERE 
         fgm.fiscal_year = 2021) total ON 1=1
WHERE 
    fgm.fiscal_year = 2021
GROUP BY 
    dc.channel, total.total_sales;
```
## Ad-Hoc Request 10: Follow-up: Get the Top 3 products in each division that have a high total_sold_quantity in the fiscal_year 2021?

```sql
SELECT
    division,
    product_code,
    product,
    total_sold_quantity
FROM (
    SELECT
        dp.division,
        fm.product_code,
        dp.product,
        SUM(fm.sold_quantity) AS total_sold_quantity,
        ROW_NUMBER() OVER (PARTITION BY dp.division ORDER BY SUM(fm.sold_quantity) DESC) AS rank_order
    FROM
        `gdb023`.`fact_sales_monthly` fm
    JOIN
        `gdb023`.`dim_product` dp ON fm.product_code = dp.product_code
    WHERE
        fm.fiscal_year = 2021
    GROUP BY
        dp.division, fm.product_code, dp.product
) AS ranked_products
WHERE
    rank_order <= 3;
```

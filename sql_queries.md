# SQL Queries Documentation

## Ad-Hoc Request 1: Provide the list of markets in which customer "Atliq Exclusive" operates its business in the APAC region

```sql
SELECT DISTINCT market
FROM gdb023.dim_customer
WHERE customer = 'Atliq Exclusive' AND region = 'APAC';
```
## Ad-Hoc Request 2: What is the percentage of unique product increase in 2021 vs. 2020?

```sql
SELECT
    COUNT(DISTINCT CASE WHEN fiscal_year = 2020 THEN product_code END) AS unique_products_2020,
    COUNT(DISTINCT CASE WHEN fiscal_year = 2021 THEN product_code END) AS unique_products_2021,
    ((COUNT(DISTINCT CASE WHEN fiscal_year = 2021 THEN product_code END) - COUNT(DISTINCT CASE WHEN fiscal_year = 2020 THEN product_code END)) 
    / COUNT(DISTINCT CASE WHEN fiscal_year = 2020 THEN product_code END)) * 100 AS percentage_chg
FROM
    fact_sales_monthly;
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
SELECT 
    d.segment,
    COUNT(DISTINCT fm.product_code) AS product_count_2020,
    (SELECT COUNT(DISTINCT fm.product_code) 
    FROM `gdb023`.`fact_sales_monthly` fm 
    WHERE fm.fiscal_year = 2021) AS product_count_2021,
    COUNT(DISTINCT fm.product_code) - (SELECT COUNT(DISTINCT fm.product_code) 
    FROM `gdb023`.`fact_sales_monthly` fm 
    WHERE fm.fiscal_year = 2020) AS difference
FROM `gdb023`.`dim_product` d
JOIN `gdb023`.`fact_sales_monthly` fm ON d.product_code = fm.product_code
GROUP BY d.segment
ORDER BY difference DESC
LIMIT 1;
```
## Ad-Hoc Request 5: Get the products that have the highest and lowest manufacturing costs. The final output should contain these fields

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
## Ad-Hoc Request 6: Generate a report which contains the top 5 customers who received an average high pre_invoice_discount_pct for the fiscal year 2021 and in the Indian market. The final output contains these fields

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
SELECT
    MONTH(fm.date) AS Month,
    YEAR(fm.date) AS Year,
    SUM(fm.sold_quantity * fp.gross_price) AS Gross_sales_Amount
FROM
    `gdb023`.`fact_sales_monthly` fm
JOIN
    `gdb023`.`fact_gross_price` fp ON fm.product_code = fp.product_code
JOIN
    `gdb023`.`dim_customer` dc ON fm.customer_code = dc.customer_code
WHERE
    dc.customer = 'Atliq Exclusive'
GROUP BY
    MONTH(fm.date), YEAR(fm.date);
```
## Ad-Hoc Request 8: In which quarter of 2020, got the maximum total_sold_quantity?

```sql
SELECT 
    QUARTER(date) AS Quarter,
    SUM(sold_quantity) AS total_sold_quantity
FROM fact_sales_monthly
WHERE YEAR(date) = 2020
GROUP BY Quarter
ORDER BY total_sold_quantity DESC
LIMIT 1;
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

  ## SQL QUERIES: 
 
 I used SQL for cleaning and Analysis. You can find below the SQL queries.

### Phase 1: DATA CLEANING

##### Table Creation followed by importing the cleaned dataset.

``` SQL
CREATE TABLE transaction_dataset (
    tr_id serial PRIMARY KEY,
    p_id TEXT,
    c_id TEXT,
    tr_date TEXT,
    online_order TEXT,
    order_status TEXT,
    brand TEXT,
    product_line TEXT,
    product_class TEXT,
    product_size TEXT,
    list_price TEXT,
    standard_cost TEXT,
    product_first_sold_date TEXT
);

```

##### Change the name of the column tr_id to transaction_id

``` SQL
ALTER TABLE transaction_dataset
RENAME COLUMN tr_id to transaction_id
```

##### Change the name of the column p_id to product_id

``` SQL
ALTER TABLE transaction_dataset
RENAME COLUMN p_id to product_id
```

##### Change the name of the column c_id to customer_id

``` SQL
ALTER TABLE transaction_dataset
RENAME COLUMN c_id to customer_id
```

##### Change the name of the column tr_date to transaction_date

``` SQL
ALTER TABLE transaction_dataset
RENAME COLUMN tr_date to transaction_date
```

##### Remove the column product_class

``` SQL
ALTER TABLE transaction_dataset
DROP COLUMN product_class
```

##### Remove the column product_size

``` SQL
ALTER TABLE transaction_dataset
DROP COLUMN product_size
```

##### Change the data type to date for transaction_date

``` SQL
ALTER TABLE transaction_dataset
ALTER COLUMN transaction_date TYPE date USING transaction_date::date
```

##### Change the data type to date for product_first_sold_date (includes conversion to date and then updating the table)

``` SQL
UPDATE transaction_dataset
SET product_first_sold_date = TO_DATE('1899-12-30', 'YYYY-MM-DD') + (product_first_sold_date || ' days')::interval;

ALTER TABLE transaction_dataset
ALTER COLUMN product_first_sold_date TYPE date USING product_first_sold_date::date
```

##### Change the data type to numeric for list_price

``` SQL
ALTER TABLE transaction_dataset
ALTER COLUMN list_price TYPE numeric USING list_price::numeric
```

##### Change the data type to numeric for standard_cost

``` SQL
ALTER TABLE transaction_dataset
ALTER COLUMN standard_cost TYPE numeric USING standard_cost::numeric
```

### Phase 2: COHORT ANALYSIS

#### Percentages speak volumes when it comes to understanding customer engagement. Calculating cohort percentages allows us to see how each cohort's size changes over time relative to its initial size. This step is crucial because it helps us gauge the effectiveness of our strategies in retaining and engaging customers. Ultimately, it guides us in optimizing our efforts to improve customer loyalty and satisfaction.

``` SQL
-- Create a temp table with all the results
CREATE TEMPORARY TABLE cohort_analysis AS 

-- Extracting YM from transaction date for approved transactions
WITH YM_Extraction AS (
 SELECT customer_id, transaction_date,
           EXTRACT(YEAR FROM transaction_date) * 100 + EXTRACT(MONTH FROM transaction_date) AS YM
    FROM (SELECT * 
FROM transaction_dataset
WHERE order_status = 'Approved') AS approved_transactions
),
-- Find the Minimum of YM 
MIN_YM AS (
 SELECT MIN(YM) AS min_ym
    FROM YM_Extraction
),
-- Find the transaction_month_index
Transaction_Month_Index AS (
	SELECT e.customer_id, e.YM-m.min_ym AS transaction_month_index
    FROM YM_Extraction AS e
	CROSS JOIN
	MIN_YM AS m
),
-- Find the cohort month
Cohort_Month AS (
SELECT customer_id, MIN(transaction_month_index) AS cohort_month
FROM Transaction_Month_Index
group by customer_id
),
-- Find the cohort index
Cohort_Index AS (
	SELECT cm.customer_id, (tmi.transaction_month_index-cm.cohort_month) AS cohort_index
FROM Transaction_Month_Index AS tmi
JOIN
Cohort_Month AS cm
ON tmi.customer_id = cm.customer_id
)
-- Select the customer_id, cohort month, cohort index and the customer count.
	SELECT cm.customer_id, cm.cohort_month, ci.cohort_index, 
COUNT(DISTINCT cm.customer_id) AS customer_count
FROM Cohort_Month AS cm
JOIN 
Cohort_Index AS ci
ON cm.customer_id = ci.customer_id
GROUP BY cm.customer_id,cm.cohort_month, ci.cohort_index

-- Pivot the above table (cohort analysis)

-- Create an extension to use crosstab for pivot

CREATE EXTENSION IF NOT EXISTS tablefunc;

-- Pivot using crosstab and store it as a temp table

CREATE TEMPORARY TABLE pivot AS
SELECT * FROM crosstab(
'SELECT cohort_month, cohort_index, SUM(customer_count) AS values
	FROM cohort_analysis
	GROUP BY cohort_month, cohort_index
	ORDER BY cohort_month, cohort_index',
	'SELECT DISTINCT cohort_index FROM cohort_analysis ORDER BY cohort_index'
) AS cohort_analysis_results(cohort_month numeric, "0" numeric,"1" numeric,"2" numeric,"3" numeric,"4" numeric,"5" numeric, "6" numeric,"7" numeric,"8" numeric,"9" numeric,"10" numeric,"11" numeric)

-- Calculate the retention rate

SELECT cohort_month,
       ROUND 100.0 * "0" / "0" AS "0",
       100.0 * "1" / "0" AS "1",
       100.0 * "2" / "0" AS "2",
       100.0 * "3" / "0" AS "3",
       100.0 * "4" / "0" AS "4",
       100.0 * "5" / "0" AS "5",
       100.0 * "6" / "0" AS "6",
       100.0 * "7" / "0" AS "7",
       100.0 * "8" / "0" AS "8",
       100.0 * "9" / "0" AS "9",
       100.0 * "10" / "0" AS "10",
       100.0 * "11" / "0" AS "11"
FROM pivot
ORDER BY Cohort_Date;
```

### Phase 3: KEY INSIGHTS

##### 1. What are the unique brands available in the dataset?

``` SQL
SELECT DISTINCT brand 
FROM transaction_dataset
```

##### 2. How many unique customers made transactions in the dataset?

``` SQL
SELECT COUNT (DISTINCT customer_id)
FROM transaction_dataset
```

##### 3. How many transactions were approved and how many were not approved?

``` SQL
SELECT
SUM(CASE WHEN order_status = 'Approved' THEN 1 ELSE 0 END) AS Approved_transactions,
SUM(CASE WHEN order_status != 'Approved' THEN 1 ELSE 0 END) AS Unapproved_transactions
FROM transaction_dataset
```

##### 4. List the top product lines with the highest average list price.

``` SQL
SELECT product_line, ROUND(AVG(list_price),2) AS avg_list_price
FROM transaction_dataset
GROUP BY product_line
ORDER BY avg_list_price desc
```

##### 5. List the product_id , list_price, standard_cost of the products where the list_price is greater than twice the standard_cost.

``` SQL
SELECT product_id, list_price, standard_cost
FROM transaction_dataset
WHERE list_price > (2*standard_cost)
```

##### 6. Calculate the average List_price for each product line.

``` SQL
SELECT product_line, ROUND(AVG(list_price),2) AS avg_list_price
FROM transaction_dataset
GROUP BY product_line
```

##### 7. Which brand has the maximum difference between the List_price and the standard_cost of their products?

``` SQL
SELECT brand, MAX(List_price - standard_cost) AS max_difference
FROM transaction_dataset
GROUP BY brand
ORDER BY max_difference DESC
LIMIT 1;
```

##### 8. List the customer_id and the count of their transactions, ordered by the number of transactions in descending order.

``` SQL
SELECT customer_id, COUNT(customer_id) AS transaction_count
FROM transaction_dataset
GROUP BY customer_id
ORDER BY transaction_count desc
```

##### 9. Find the total sales amount for each brand, sorted in descending order of total sales.

``` SQL
SELECT brand, SUM(list_price) as total_sales
FROM transaction_dataset
GROUP BY brand
ORDER BY total_sales DESC
```

##### 10. What are the top 5 products with the highest profit margin?

``` SQL
SELECT DISTINCT product_id, brand, product_line, (list_price)-(standard_cost) as profit_margin
FROM transaction_dataset
ORDER BY profit_margin DESC
LIMIT 5
```

##### 11. For each customer, find the total number of transactions, the total amount they spent, and their average profit per transaction.

``` SQL
SELECT customer_id,
COUNT(transaction_id) AS total_transactions,
SUM(list_price) AS total_amount_spent,
ROUND((AVG(list_price)-AVG(standard_cost)),2) AS average_profit_per_transaction
FROM transaction_dataset
GROUP BY customer_id
```

##### 12. Find the top 5 product lines with the highest total revenue and their percentage contribution to the overall revenue.

``` SQL
WITH Total_Revenue_per_product AS (
SELECT product_line, SUM(list_price) AS total_revenue_per_product
	FROM transaction_dataset
	GROUP BY product_line
),
Total_Revenue AS (
SELECT SUM(list_price) AS total_revenue
	FROM transaction_dataset
)
SELECT product_line, total_revenue, 
	ROUND((trp.total_revenue_per_product * 100 / tr.total_revenue),2) AS percentage_revenue_contribution
	FROM Total_Revenue_per_product AS trp
	JOIN
	Total_Revenue AS tr
	ON 1=1
	ORDER BY trp.total_revenue_per_product DESC
	LIMIT 5
```

##### 13. Identify the customers who have made at least one transaction for each product line available.

``` SQL
SELECT customer_id, COUNT(DISTINCT product_line) AS total_product_line
FROM transaction_dataset
GROUP BY customer_id
HAVING COUNT(DISTINCT product_line) = (SELECT COUNT(DISTINCT product_line) FROM transaction_dataset)
```


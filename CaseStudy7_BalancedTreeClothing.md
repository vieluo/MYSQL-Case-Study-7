# 8 Weeks SQL Challenge - 7th Week

### This [case study](https://8weeksqlchallenge.com/case-study-7/)  is provided by Danny Ma

## Introduction
Balanced Tree Clothing Company prides themselves on providing an optimised range of clothing and lifestyle wear for the modern adventurer!

Danny, the CEO of this trendy fashion company has asked you to assist the team’s merchandising teams analyse their sales performance and generate a basic financial report to share with the wider business.

The database can be found [HERE](https://8weeksqlchallenge.com/case-study-7/)

## High Level Sales Analysis
1. What was the total quantity sold for all products?

```sql
SELECT SUM(qty) AS total_sold
FROM sales
```

| total_sold |
|------------|
| 45216      |

2. What is the total generated revenue for all products before discounts?

```sql
SELECT SUM((price+discount)*qty) AS revenue_before_discount
FROM sales
```

| revenue_before_discount |
|------------|
| 1835884     |

3. What was the total discount amount for all products?

```sql
SELECT SUM(discount*qty) AS total_discount
FROM sales
```

| total_discount |
|------------|
| 546431     |


## Transaction Analysis
1. How many unique transactions were there?

```sql
SELECT COUNT(DISTINCT txn_id) AS count_transaction
FROM sales
```

| count_transaction |
|------------|
| 2500    |

2. What is the average unique products purchased in each transaction?

```sql
SELECT ROUND(AVG(distinct_product),2) AS avg_distict_product_per_transaction
FROM (
SELECT txn_id, COUNT(DISTINCT prod_id) AS distinct_product
FROM balanced_tree.sales
GROUP BY txn_id
) temp
```

| avg_distict_product_per_transaction |
|------------|
| 6.04    |

3. What are the 25th, 50th and 75th percentile values for the revenue per transaction?

```sql
WITH temp1 AS (
SELECT txn_id, SUM(price*qty) AS revenue_per_transaction, ROW_NUMBER() OVER (ORDER BY SUM(price*qty)) AS row_num 
FROM sales
GROUP BY txn_id
ORDER BY SUM(price)
),

temp2 AS (
SELECT ROUND(MAX(row_num)*25/100) AS 25_percentile, ROUND(MAX(row_num)*50/100) AS 50_percentile, ROUND(MAX(row_num)*75/100) AS 75_percentile
FROM temp1
)

SELECT temp1.*, temp2.25_percentile
FROM temp1, temp2
WHERE row_num = temp2.25_percentile /*Repeat the process to get 50 and 75 percentile, change the numer to 50 and 70*/
```

|   txn_id | revenue_per_transaction | row_num | 25_percentile  |
|----------|-------------------------|---------|----------------|
| a23d26   | 375                     | 625     | 625            | 


|   txn_id | revenue_per_transaction | row_num | 50_percentile  |
|----------|-------------------------|---------|----------------|
| 93b088   | 509                     | 1250     | 1250            | 


|   txn_id | revenue_per_transaction | row_num | 70_percentile  |
|----------|-------------------------|---------|----------------|
| e3a7a2   | 647                     | 1875     | 1875            | 

4. What is the average discount value per transaction?

```sql
SELECT ROUND(AVG(disount_by_transaction),2) AS avg_disount_by_transaction
FROM (
SELECT txn_id, SUM(discount*qty) AS disount_by_transaction
FROM sales
GROUP BY txn_id
)temp
```

| avg_disount_by_transaction |
|------------|
| 218.57     |

5. What is the percentage split of all transactions for members vs non-members?

```sql
WITH transaction_member AS (
SELECT COUNT(DISTINCT txn_id) AS amount_transcation_member
FROM sales
WHERE member = 1
),
transaction_non_member AS (
SELECT COUNT(DISTINCT txn_id) AS amount_transcation_non_member
FROM sales
WHERE member = 0
)

SELECT ROUND(transaction_member.amount_transcation_member/2500*100,2) /*the total amount of transaction we get in the Q1*/ AS percentage_transaction_member,
ROUND(transaction_non_member.amount_transcation_non_member/2500*100,2) AS percentage_transaction_non_member

FROM transaction_member,transaction_non_member
```

|   percentage_transaction_member | percentage_transaction_non_member  |
|---------------------------------|------------------------------------|
| 60.20                           | 39.80                              | 

6. What is the average revenue for member transactions and non-member transactions?

```sql
SELECT ROUND(AVG(price*qty),2) AS revenue_member
FROM sales
WHERE member = 1
```

| revenue_member |
|------------|
| 85.75      |

```sql
SELECT ROUND(AVG(price*qty),2) AS revenue_non_member
FROM sales
WHERE member = 0
```

| revenue_non_member |
|------------|
| 84.93      |

## Product Analysis

1. What are the top 3 products by total revenue before discount?

```sql
WITH before_discount AS (
SELECT prod_id, SUM(price_before_discount) AS sum_price_before_discount
FROM (
SELECT *, (price + discount)*qty AS price_before_discount 
FROM balanced_tree.sales
) temp
GROUP BY prod_id
)
SELECT before_discount.prod_id, before_discount.sum_price_before_discount, product_details.product_name
FROM before_discount
JOIN product_details
ON before_discount.prod_id = product_details.product_id
ORDER BY sum_price_before_discount desc
LIMIT 3
```

| # prod_id | sum_price_before_discount | product_name                  |
|-----------|---------------------------|-------------------------------|
| 2a2353    | 264734                    | Blue Polo Shirt - Mens        |
| 9ec847    | 256326                    | Grey Fashion Jacket - Womens  |
| 5d267b    | 197944                    | White Tee Shirt - Mens        |

2. What is the total quantity, revenue and discount for each segment?

```sql
WITH sales_details AS (
SELECT product_id, product_details.price, product_name, category_id, segment_id, style_id, category_name, segment_name, style_name, qty, discount, member, txn_id, start_txn_time
FROM sales
JOIN product_details
ON sales.prod_id = product_details.product_id
)

SELECT segment_id, segment_name, SUM(qty) AS total_qty_seg, SUM(discount) AS total_discount_seg, SUM(qty*price) AS total_revenue_seg
FROM sales_details
GROUP BY  segment_name, segment_id
```

| segment_id | segment_name | total_qty_seg | total_discount_seg | total_revenue_seg  |
|------------|--------------|---------------|--------------------|--------------------|
| 3          | Jeans        | 11349         | 45740              | 208350             |
| 5          | Shirt        | 11265         | 46043              | 406143             |
| 6          | Socks        | 11217         | 45465              | 307977             |
| 4          | Jacket       | 11385         | 45452              | 366983             |

3. What is the top selling product for each segment?

```sql
sales_segment_product AS (
SELECT product_id, product_name, segment_id, segment_name, SUM(qty) AS sales, ROW_NUMBER() OVER (PARTITION BY segment_id ORDER BY SUM(qty)) AS sales_number
FROM sales_details /*we are using the same sales_details table in Q2 */
GROUP BY  product_id, product_name, segment_id, segment_name
) /* order the best sales in each segment */

SELECT product_name, segment_name, sales
FROM sales_segment_product
WHERE sales_number = 3
```

| product_name                  | segment_name | sales  |
|-------------------------------|--------------|--------|
| Navy Oversized Jeans - Womens | Jeans        | 3856   |
| Grey Fashion Jacket - Womens  | Jacket       | 3876   |
| Blue Polo Shirt - Mens        | Shirt        | 3819   |
| Navy Solid Socks - Mens       | Socks        | 3792   |

4. What is the total quantity, revenue and discount for each category?

```sql
SELECT category_id, category_name, SUM(qty) AS total_qty_cat, SUM(discount) AS total_discount_cat, SUM(qty*price) AS total_revenue_cat
FROM sales_details  /*we are using the same sales_details table in Q2 */
GROUP BY  category_id, category_name
```

| category_id | category_name | total_qty_cat | total_discount_cat | total_revenue_cat  |
|-------------|---------------|---------------|--------------------|--------------------|
| 1           | Womens        | 22734         | 91192              | 575333             |
| 2           | Mens          | 22482         | 91508              | 714120             |

5. What is the top selling product for each category?

```sql
sales_cat_product AS (
SELECT product_id, product_name, category_id, category_name, SUM(qty) AS sales, ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY SUM(qty)) AS sales_number
FROM sales_details  /*we are using the same sales_details table in Q2 */
GROUP BY  product_id, product_name, category_id, category_name
) /* order the best sales in each category */

SELECT product_name, category_name, sales
FROM sales_cat_product
WHERE sales_number = 6
```

| product_name                 | category_name | sales  |
|------------------------------|---------------|--------|
| Grey Fashion Jacket - Womens | Womens        | 3876   |
| Blue Polo Shirt - Mens       | Mens          | 3819   |

6. What is the percentage split of revenue by product for each segment?

```sql
sales_segment AS (
SELECT product_id, product_name, segment_id, segment_name, SUM(qty*price) AS sales
FROM sales_details /*we are using the same sales_details table in Q2 */
GROUP BY  product_id, product_name, segment_id, segment_name
ORDER BY segment_id, SUM(qty*price) 
)
SELECT *, SUM(sales) OVER (PARTITION BY segment_id) AS total_sales_seg, ROUND((sales/SUM(sales) OVER (PARTITION BY segment_id))*100,2) AS percentage_pro_in_seg
FROM sales_segment
```

| product_id | product_name                     | segment_id | segment_name | sales  | total_sales_seg | percentage_pro_in_seg  |
|------------|----------------------------------|------------|--------------|--------|-----------------|------------------------|
| e31d39     | Cream Relaxed Jeans - Womens     | 3          | Jeans        | 37070  | 208350          | 17.79                  |
| c4a632     | Navy Oversized Jeans - Womens    | 3          | Jeans        | 50128  | 208350          | 24.06                  |
| e83aa3     | Black Straight Jeans - Womens    | 3          | Jeans        | 121152 | 208350          | 58.15                  |
| 72f5d4     | Indigo Rain Jacket - Womens      | 4          | Jacket       | 71383  | 366983          | 19.45                  |
| d5e9a6     | Khaki Suit Jacket - Womens       | 4          | Jacket       | 86296  | 366983          | 23.51                  |
| 9ec847     | Grey Fashion Jacket - Womens     | 4          | Jacket       | 209304 | 366983          | 57.03                  |
| c8d436     | Teal Button Up Shirt - Mens      | 5          | Shirt        | 36460  | 406143          | 8.98                   |
| 5d267b     | White Tee Shirt - Mens           | 5          | Shirt        | 152000 | 406143          | 37.43                  |
| 2a2353     | Blue Polo Shirt - Mens           | 5          | Shirt        | 217683 | 406143          | 53.60                  |
| b9a74d     | White Striped Socks - Mens       | 6          | Socks        | 62135  | 307977          | 20.18                  |
| 2feb6b     | Pink Fluro Polkadot Socks - Mens | 6          | Socks        | 109330 | 307977          | 35.50                  |
| f084eb     | Navy Solid Socks - Mens          | 6          | Socks        | 136512 | 307977          | 44.33                  |

7. What is the percentage split of revenue by segment for each category?

```sql
sales_cat AS (
SELECT category_id, category_name, segment_id, segment_name, SUM(qty*price) AS sales
FROM sales_details /*we are using the same sales_details table in Q2 */
GROUP BY  category_id, category_name, segment_id, segment_name
ORDER BY category_id, SUM(qty*price) 
)
SELECT *, SUM(sales) OVER (PARTITION BY category_id) AS total_sales_cat, ROUND((sales/SUM(sales) OVER (PARTITION BY category_id))*100,2) AS percentage_seg_in_cat
FROM sales_cat
```

| category_id | category_name | segment_id | segment_name | sales  | total_sales_cat | percentage_seg_in_cat  |
|-------------|---------------|------------|--------------|--------|-----------------|------------------------|
| 1           | Womens        | 3          | Jeans        | 208350 | 575333          | 36.21                  |
| 1           | Womens        | 4          | Jacket       | 366983 | 575333          | 63.79                  |
| 2           | Mens          | 6          | Socks        | 307977 | 714120          | 43.13                  |
| 2           | Mens          | 5          | Shirt        | 406143 | 714120          | 56.87                  |

8. What is the percentage split of total revenue by category?

```sql
sales_cat AS (
SELECT category_id, category_name, SUM(qty*price) AS sales
FROM sales_details /*we are using the same sales_details table in Q2 */
GROUP BY  category_id, category_name
ORDER BY category_id, SUM(qty*price) 
)

SELECT *, ROUND((sales/SUM(sales) OVER())*100,2) AS percentage_cat
FROM sales_cat
GROUP BY category_id, category_name
```

| category_id | category_name | sales  | percentage_cat  |
|-------------|---------------|--------|-----------------|
| 1           | Womens        | 575333 | 44.62           |
| 2           | Mens          | 714120 | 55.38           |

9. What is the total transaction “penetration” for each product? (hint: penetration = number of transactions where at least 1 quantity of a product was purchased divided by total number of transactions)

```sql
SELECT DISTINCT prod_id, product_name, transaction, total_transaction, penetration
FROM (
SELECT * , count(prod_id) OVER(PARTITION BY prod_id) AS transaction, count(prod_id) OVER() AS total_transaction,
ROUND ((count(prod_id) OVER(PARTITION BY prod_id)) / (count(prod_id) OVER()) *100,2)AS penetration
FROM balanced_tree.sales
) temp
JOIN product_details
ON prod_id = product_id
```

| prod_id | product_name                     | transaction | total_transaction | penetration  |
|---------|----------------------------------|-------------|-------------------|--------------|
| c4a632  | Navy Oversized Jeans - Womens    | 1274        | 15095             | 8.44         |
| e83aa3  | Black Straight Jeans - Womens    | 1246        | 15095             | 8.25         |
| e31d39  | Cream Relaxed Jeans - Womens     | 1243        | 15095             | 8.23         |
| d5e9a6  | Khaki Suit Jacket - Womens       | 1247        | 15095             | 8.26         |
| 72f5d4  | Indigo Rain Jacket - Womens      | 1250        | 15095             | 8.28         |
| 9ec847  | Grey Fashion Jacket - Womens     | 1275        | 15095             | 8.45         |
| 5d267b  | White Tee Shirt - Mens           | 1268        | 15095             | 8.40         |
| c8d436  | Teal Button Up Shirt - Mens      | 1242        | 15095             | 8.23         |
| 2a2353  | Blue Polo Shirt - Mens           | 1268        | 15095             | 8.40         |
| f084eb  | Navy Solid Socks - Mens          | 1281        | 15095             | 8.49         |
| b9a74d  | White Striped Socks - Mens       | 1243        | 15095             | 8.23         |
| 2feb6b  | Pink Fluro Polkadot Socks - Mens | 1258        | 15095             | 8.33         |

10. What is the most common combination of at least 1 quantity of any 3 products in a 1 single transaction?


```sql
WITH temp AS (
SELECT qty,sales.prod_id,member,txn_id,start_txn_time,product_name
FROM sales
JOIN product_details
ON sales.prod_id = product_details.product_id
)
SELECT CONCAT_WS(',', cp1.product_name, cp2.product_name, cp3.product_name) AS ProductCombo, count(*) AS cnt
FROM temp cp1
JOIN temp cp2 ON cp1.txn_id = cp2.txn_id AND cp1.product_name < cp2.product_name
JOIN temp cp3 ON cp2.txn_id = cp3.txn_id AND cp2.product_name < cp3.product_name 
/* Permutation of 3 product */
GROUP BY ProductCombo
ORDER BY cnt DESC
LIMIT 1
```

| # ProductCombo                                                                  | cnt  |
|---------------------------------------------------------------------------------|------|
| Grey Fashion Jacket - Womens,Teal Button Up Shirt - Mens,White Tee Shirt - Mens | 352  |

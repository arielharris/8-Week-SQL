# Week #1 - Danny's Diner
For the complete case study information, please visit [Data with Danny](https://8weeksqlchallenge.com/case-study-1/).

## The Challenge
In this case study we are tasked with using three datasets to find information about the patterns of customers of Danny's Diner. Information such as their visiting patterns, ordering habits, and loyalty enrollment will all be investigated. Below is the entity relationship provided by Data with Danny.

![image](https://github.com/arielharris/8-Week-SQL/assets/54371185/27c68ccb-1473-4f20-9e7d-377a41831a98)

## The Questions
Data with Danny includes a DB Fiddle to to execute code, however I felt more comfortable using Ubuntu on my personal desktop. All answers are found using PostgreSQL.

1. What is the total amount each customer spent at the restaurant?
```sql
SELECT sales.customer_id, SUM(menu.price) AS total_spent
FROM sales JOIN menu on sales.product_id = menu.product_id
GROUP BY sales.customer_id ORDER BY sales.customer_id ASC;
```
The first step in this solution was to join the menu and sales tables as we needed every order a customer had placed as well as the cost for each item.
After the join we needed to select and group by the customer_id and the total for the prices of each item they ordered (`SUM(menu.pric)`). Finally ordering by the customer_id in ascending order to make the outcome table visually easy to read. 

| customer_id  | total_spent |
| ------------- | ------------- |
| A  | 76  |
| B  | 74  |
| C  | 36  |

2. How many days has each customer visited the reastaurant?
```sql
SELECT customer_id, COUNT(DISTINCT order_date) FROM sales GROUP BY customer_id
```
All of this information can be pulled from just the sales table, it is just important to remember there are some repeat days for the same customer so DISTINCT must be used.

| customer_id  | count |
| ------------- | ------------- |
| A  | 4  |
| B  | 6  |
| C  | 2  |

3. What was the first item form the menu purchased by each customer?
```sql
SELECT sales.customer_id, menu.product_name as first_ordered FROM sales JOIN menu ON sales.product_id = menu.product_id WHERE sales.order_date=(SELECT MIN(sales.order_date) FROM sales)
ORDER BY customer_id ASC;
```


| customer_id  | first_ordered |
| ------------- | ------------- |
| A  | sushi  |
| A  | curry  |
| B  | curry  |
| C  | ramen  |
| C  | ramen  |

4. What is the most purchased item on the menu and how many times was it purchased by all customers?
```sql
SELECT menu.product_name, COUNT(sales.product_id) as total_orders
FROM sales JOIN menu ON sales.product_id=menu.product_id
GROUP BY menu.product_name ORDER BY total_orders DESC LIMIT 1;
```

| product_name  | total_orders |
| ------------- | ------------- |
| ramen  | 8  |

5. Which item was the most popular for each customer?
```sql
WITH most_ordered AS (
SELECT sales.customer_id, menu.product_name, COUNT(sales.product_id) as ord_count, DENSE_RANK() OVER( PARTITION BY customer_id ORDER BY COUNT (customer_id)DESC) as rank FROM menu join sales on menu.product_id = sales.product_id GROUP BY sales.customer_id, menu.product_name)
SELECT customer_id, product_name, ord_count FROM most_ordered WHERE rank = 1;
```

| customer_id | product_name | ord_count|
|-------------|--------------|-----------|
| A           | ramen        |         3|
| B           | sushi        |         2|
| B           | curry        |         2|
| B           | ramen        |         2|
| C           | ramen        |         3|

6. Which item was purchased first by the customer after they became a member?
```sql
WITH next_date AS (
SELECT sales.customer_id, sales.order_date, sales.product_id, members.join_date, DENSE_RANK() OVER(PARTITION BY sales.customer_id ORDER BY sales.order_date) as rank FROM sales JOIN members ON sales.customer_id=members.customer_id WHERE sales.order_date > members.join_date GROUP BY sales.customer_id, sales.order_date, members.join_date, sales.product_id)

SELECT next_date.customer_id, next_date.join_date, next_date.order_date, menu.product_name FROM menu JOIN next_date ON menu.product_id=next_date.product_id WHERE rank = 1 ORDER BY next_date.customer_id ASC;
```
|customer_id | join_date  | order_date | product_name|
|-------------|------------|------------|--------------|
| A           | 2021-01-07 | 2021-01-10 | ramen|
| B           | 2021-01-09 | 2021-01-11 | sushi|

7. Which item was purchased just before the customer became a member?
```sql
WITH prev_date AS (
SELECT sales.customer_id, sales.order_date, sales.product_id, members.join_date, DENSE_RANK() OVER(PARTITION BY sales.customer_id ORDER BY sales.order_date DESC) as rank FROM sales JOIN members ON sales.customer_id=members.customer_id WHERE sales.order_date < members.join_date GROUP BY sales.customer_id, sales.order_date, members.join_date, sales.product_id)
SELECT prev_date.customer_id, prev_date.join_date, prev_date.order_date, menu.product_name FROM menu JOIN prev_date ON menu.product_id=prev_date.product_id WHERE rank = 1 ORDER BY prev_date.customer_id ASC;
```
 |customer_id | join_date  | order_date | product_name|
|-------------|------------|------------|--------------|
| A           | 2021-01-07 | 2021-01-01 | sushi|
| A           | 2021-01-07 | 2021-01-01 | curry|
| B           | 2021-01-09 | 2021-01-04 | sushi|

8. What is the total items and amount spent for each member before they became a member?
```sql
WITH prev_date AS (
SELECT sales.customer_id, sales.product_id, sales.order_date FROM sales JOIN members ON sales.customer_id=members.customer_id WHERE sales.order_date < members.join_date GROUP BY sales.customer_id, sales.product_id, sales.order_date)
SELECT prev_date.customer_id, SUM(menu.price) as total_spent, COUNT(prev_date.product_id) as total_items FROM menu JOIN prev_date ON menu.product_id=prev_date.product_id GROUP BY prev_date.customer_id ORDER BY prev_date.customer_id ASC;
```
| customer_id  | total_spent | total_items|
| ------------- | ------------- | ------------- |
| A | 25 | 2 |
| B | 40 | 3 |

9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
```sql
WITH point_asg AS (
SELECT menu.product_id, CASE WHEN product_name = 'sushi' THEN price*20 ELSE price*10 END AS points FROM menu)
SELECT sales.customer_id, SUM(point_asg.points) as total_points FROM sales JOIN point_asg ON sales.product_id = point_asg.product_id GROUP BY sales.customer_id ORDER BY customer_id ASC;
```
|customer_id | total_points|
| ------------- | ------------- |
| A | 860 |
| B | 940 | 
| C | 360 |

10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
```sql
WITH date_range AS (
SELECT customer_id, join_date, join_date+6 as end_promo_date, DATE_TRUNC('month','2021-01-31'::DATE) + interval '1 month' - interval '1 day' AS last_date FROM members)
SELECT sales.customer_id, SUM(CASE WHEN product_name = 'sushi' THEN menu.price*20 WHEN sales.order_date between date_range.join_date AND date_range.end_promo_date THEN menu.price*20 ELSE menu.price*10 END) AS total_points
FROM sales JOIN date_range ON sales.customer_id = date_range.customer_id AND sales.order_date <= date_range.last_date JOIN menu ON sales.product_id = menu.product_id GROUP BY sales.customer_id;
```
|customer_id | total_points|
| ------------- | ------------- |
| A | 1370 |
| B | 820 | 

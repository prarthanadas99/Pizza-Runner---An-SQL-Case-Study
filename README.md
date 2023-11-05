# Pizza-Runner---An-SQL-Case-Study

![Alt text](https://8weeksqlchallenge.com/images/case-study-designs/2.png)

### Introduction
Did you know that over 115 million kilograms of pizza is consumed daily worldwide??? (Well according to Wikipedia anyway…)
Danny was scrolling through his Instagram feed when something really caught his eye - “80s Retro Styling and Pizza Is The Future!”
Danny was sold on the idea, but he knew that pizza alone was not going to help him get seed funding to expand his new Pizza Empire - so he had one more genius idea to combine with it - he was going to Uberize it - and so Pizza Runner was launched!
Danny started by recruiting “runners” to deliver fresh pizza from Pizza Runner Headquarters (otherwise known as Danny’s house) and also maxed out his credit card to pay freelance developers to build a mobile app to accept orders from customers.

**A. Pizza Metrics**

**1. How many pizzas were ordered?**
```sql
select count(order_id) as Total_Pizza_Orders from customer_orders;
```
**2. How many unique customer orders were made?**
```sql
select count(distinct order_id) as Unique_Customer_Orders from customer_orders;
```
**3. How many successful orders were delivered by each runner?**
```sql
select runner_id, count(order_id) as total_Orders_by_each_runner from runner_orders
where cancellation is null 
group by runner_id;
```
**4. How many of each type of pizza was delivered?**
```sql
select p.pizza_id, p.pizza_name, count(p.pizza_name) as pizza_delivered from pizza_names p
inner join customer_orders c on p.pizza_id = c.pizza_id
group by p.pizza_id, p.pizza_name;
```
**5. How many of each type of pizza was delivered?**
```sql
select c.customer_id, p.pizza_name, count(p.pizza_name) as pizza_types_delivered from customer_orders c
inner join pizza_names p on p.pizza_id = c.pizza_id
group by c.customer_id, p.pizza_name 
order by c.customer_id;
```
**6. What was the maximum number of pizzas delivered in a single order?**
```sql
select customer_id, order_id, count(pizza_id) as number_of_pizzas from customer_orders
group by customer_id, order_id
order by 3 desc
limit 1;
```
**7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?**
```sql
select customer_id,
sum(case when exclusions is not null or extras is not null then 1 else 0 end) as pizza_change,
sum(case when exclusions is null and extras is null then 1 else 0 end) as pizza_no_change
from customer_orders
INNER JOIN runner_orders USING (order_id)
group by 1;
```
**8. How many pizzas were delivered that had both exclusions and extras?**
```sql
SELECT customer_id,
       SUM(CASE WHEN exclusions IS NOT NULL AND extras IS NOT NULL THEN 1 ELSE 0 END) AS both_change_in_pizza
FROM customer_orders
INNER JOIN runner_orders USING (order_id)
WHERE cancellation IS NULL
GROUP BY customer_id
ORDER BY customer_id;
```
**9. What was the total volume of pizzas ordered for each hour of the day?**
```sql
select customer_id, extract(hour from order_time) as order_hour, count(order_id) as order_count
from customer_orders
group by 1, 2;
```
**10. What was the volume of orders for each day of the week?**
```sql
select customer_id, dayofweek(order_time) as order_day, count(order_id) as order_count
from customer_orders
group by 1, 2;
```
**B. Runner and Customer Experience**

**1. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)**
```sql
select  extract(week from registration_date) as registration_week, count(runner_id) as total_number_of_runs
from runners
group by 1;
```
**2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order**
```sql
select ro.runner_id, round(avg(TIMESTAMPDIFF(MINUTE, co.order_time, ro.pickup_time)), 2) avg_runner_pickup_time
FROM runner_orders ro 
inner join customer_orders co on ro.order_id = co.order_id
where cancellation is null
group by 1;
```
**3. What was the average distance travelled for each customer?**
```sql
select co.customer_id, round(avg(ro.distance)) as avg_distance from customer_orders co
inner join runner_orders ro on co.order_id = ro.order_id
WHERE cancellation IS NULL
group by 1;
```
**4. What was the difference between the longest and shortest delivery times for all orders?**
```sql
select max(duration) - min(duration) as difference from runner_orders;
```
**5. What was the average speed for each runner for each delivery and do you notice any trend for these values?**
```sql
SELECT runner_id, distance AS distance_km, round(duration/60, 2) AS duration_hr, round(distance*60/duration, 2) AS average_speed
FROM runner_orders
WHERE cancellation IS NULL
ORDER BY runner_id;
```
**6. What is the successful delivery percentage for each runner?**
```sql
select runner_id, COUNT(pickup_time) AS delivered_orders, count(order_id) as orders_delivered,
ROUND(100 * COUNT(pickup_time) / COUNT(*)) AS delivery_success_percentage from runner_orders ro
group by 1;
```
```sql
CREATE
TEMPORARY TABLE row_split_customer_orders_temp AS
SELECT t.row_num, t.order_id, t.customer_id, t.pizza_id, trim(j1.exclusions) AS exclusions, trim(j2.extras) AS extras, t.order_time
FROM (SELECT *,
          row_number() over() AS row_num FROM customer_orders) t
INNER JOIN json_table(trim(replace(json_array(t.exclusions), ',', '","')),
                      '$[*]' columns (exclusions varchar(50) PATH '$')) j1
INNER JOIN json_table(trim(replace(json_array(t.extras), ',', '","')),
                      '$[*]' columns (extras varchar(50) PATH '$')) j2 ;
SELECT * FROM row_split_customer_orders_temp;

CREATE
TEMPORARY TABLE row_split_pizza_recipes_temp AS
SELECT t.pizza_id,
       trim(j.topping) AS topping_id
FROM pizza_recipes t
JOIN json_table(trim(replace(json_array(t.toppings), ',', '","')),
                '$[*]' columns (topping varchar(50) PATH '$')) j ;

SELECT * FROM row_split_pizza_recipes_temp;

CREATE
TEMPORARY TABLE standard_ingredients AS
SELECT pizza_id,pizza_name, group_concat(DISTINCT topping_name) 'standard_ingredients'
FROM row_split_pizza_recipes_temp
INNER JOIN pizza_names USING (pizza_id)
INNER JOIN pizza_toppings USING (topping_id)
GROUP BY 1,pizza_name
ORDER BY pizza_id;
```
**C. Ingredient Optimisation**

**1. What are the standard ingredients for each pizza?**
```sql
SELECT * FROM standard_ingredients;
```
**2. What was the most commonly added extra?**
```sql
WITH cte AS
  (SELECT trim(extras) AS extra_topping, count(*) AS purchase_counts
   FROM row_split_customer_orders_temp 
   WHERE extras IS NOT NULL
   GROUP BY extras)
SELECT topping_name, purchase_counts FROM cte
INNER JOIN pizza_toppings ON cte.extra_topping = pizza_toppings.topping_id
LIMIT 1;
```
**3. What was the most common exclusion?**
```sql
WITH extra_count_cte AS
  (SELECT trim(exclusions) AS extra_topping,
          count(*) AS purchase_counts
   FROM row_split_customer_orders_temp
   WHERE exclusions IS NOT NULL
   GROUP BY exclusions)
SELECT topping_name,
       purchase_counts
FROM extra_count_cte
INNER JOIN pizza_toppings ON extra_count_cte.extra_topping = pizza_toppings.topping_id
LIMIT 1;
```
**D. Pricing and Ratings**

**1. If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?**
```sql
select sum(case when co.pizza_id = 1 then 12 else 10 end) as revenue from customer_orders co
inner join runner_orders ro on ro.order_id = co.order_id
where ro.cancellation is null;
```
**2. What if there was an additional $1 charge for any pizza extras? Add cheese is $1 extra**
```sql
select  sum(case when co.pizza_id = 1 then 12 else 10 end)  as revenue from customer_orders co
inner join runner_orders ro on ro.order_id = co.order_id
where ro.cancellation is null;
```
**3. The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.
CREATE TABLE runner_rating (order_id INTEGER, rating INTEGER, review VARCHAR(100)) ;
Order 6 and 9 were cancelled**
```sql
INSERT INTO runner_rating
VALUES ('1', '1', 'Really bad service'),
       ('2', '1', NULL),
       ('3', '4', 'Took too long...'),
       ('4', '1','Runner was lost, delivered it AFTER an hour. Pizza arrived cold' ),
       ('5', '2', 'Good service'),
       ('7', '5', 'It was great, good service and fast'),
       ('8', '2', 'He tossed it on the doorstep, poor service'),
       ('10', '5', 'Delicious!, he delivered it sooner than expected too!');
```

**4. If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled - how much money does Pizza Runner have left over after these deliveries?**
```sql
select ro.runner_id, ro.order_id, ro.distance, sum(case when co.pizza_id = 1 then 12 else 10 end) as revenue,
ro.distance * 0.30 as delivery_cost from customer_orders co
inner join runner_orders ro on ro.order_id = co.order_id
where ro.cancellation is null
group by ro.runner_id, ro.order_id, ro.distance
order by ro.runner_id;
```

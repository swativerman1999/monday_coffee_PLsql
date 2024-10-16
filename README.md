# monday_coffee_PLsql

-- Q.1 Coffee Consumers Count
-- How many people in each city are estimated to consume coffee, given that 25% of the population does?	
select city_name, round((population*0.25)/1000000,2) as coffee_consumer, city_rank
from city
order by 2 desc

-- -- Q.2
-- Total Revenue from Coffee Sales
-- What is the total revenue generated from coffee sales across all cities in the last quarter of 2023?

select ci.city_name, sum(total) as revenue
from sales s
join customers c
on c.customer_id = s.customer_id
join city ci
on c.city_id=ci.city_id
where extract(quarter from sale_date)=4 AND extract(year from sale_date) = 2023
group by 1
order by 2 desc

-- Q.3
-- Sales Count for Each Product
-- How many units of each coffee product have been sold?
select p.product_name, count(s.sale_id)
from products p
join sales s
on p.product_id = s.product_id
group by 1
order by 2 desc

-- Q.4
-- Average Sales Amount per City
-- What is the average sales amount per customer in each city?
--total sales divided by no. of unique customers in each city(group by city_name)

select ci.city_name, sum(total) as revenue, count(distinct c.customer_id), 
round(sum(total)::numeric/count(distinct c.customer_id)::numeric,2) as avg_per_cust
from sales s
join customers c
on c.customer_id = s.customer_id
join city ci
on c.city_id=ci.city_id
group by 1
order by 2 desc

-- -- Q.5
-- City Population and Coffee Consumers (25%), provide a list of cities along with their populations and estimated coffee consumers.
-- return city_name, total current customers(unique customers), estimated coffee consumers (25%)

Select ci.city_name,round((ci.population*0.25)/1000000,2) as coffee_consumer, count(distinct cus.customer_id) as unique_cx
from sales s
join customers cus
on s.customer_id = cus.customer_id
join city ci
on cus.city_id=ci.city_id
Group by 1,2

--OR

WITH city_table as 
(
	SELECT 
		city_name,
		ROUND((population * 0.25)/1000000, 2) as coffee_consumers
	FROM city
),
customers_table
AS
(
	SELECT 
		ci.city_name,
		COUNT(DISTINCT c.customer_id) as unique_cx
	FROM sales as s
	JOIN customers as c
	ON c.customer_id = s.customer_id
	JOIN city as ci
	ON ci.city_id = c.city_id
	GROUP BY 1
)
SELECT 
	customers_table.city_name,
	city_table.coffee_consumers as coffee_consumer_in_millions,
	customers_table.unique_cx
FROM city_table
JOIN 
customers_table
ON city_table.city_name = customers_table.city_name

-- -- Q6
-- Top Selling Products by City
-- What are the top 3 selling products in each city based on sales volume?

select * 
from 

(	
select ci.city_name, p.product_id, count(s.sale_id) as total_oders,
DENSE_RANK() OVER(PARTITION BY ci.city_name ORDER BY COUNT(s.sale_id)DESC) as rank
from products p
join sales s 
on p.product_id = s.product_id
join customers cus
on s.customer_id=cus.customer_id
join city ci
on cus.city_id=ci.city_id
group by 1,2
order by 1,3 desc

) as t1
Where rank<=3

-- Q.7
-- Customer Segmentation by City
-- How many unique customers are there in each city who have purchased coffee products?

SELECT ci.city_name,
	COUNT(DISTINCT c.customer_id) as unique_cx
FROM sales as s
JOIN customers as c
ON c.customer_id = s.customer_id
JOIN city as ci
ON ci.city_id = c.city_id
GROUP BY 1

--OR

SELECT ci.city_name,
	COUNT(DISTINCT c.customer_id) as unique_cx
FROM city as ci
LEFT JOIN customers as c
ON ci.city_id = c.city_id
JOIN sales as s
ON s.customer_id = c.customer_id
where s.product_id In (1,2,3,4,5,6,7,8,9,10,11,12,13,14)
GROUP BY 1

-- -- Q.8
-- Average Sale vs Rent
-- Find each city and their average sale per customer and avg rent per customer

WITH city_table
AS
(
	SELECT 
		ci.city_name,
		SUM(s.total) as total_revenue,
		COUNT(DISTINCT s.customer_id) as total_cx,
		ROUND(
				SUM(s.total)::numeric/
					COUNT(DISTINCT s.customer_id)::numeric
				,2) as avg_sale_pr_cx
		
	FROM sales as s
	JOIN customers as c
	ON s.customer_id = c.customer_id
	JOIN city as ci
	ON ci.city_id = c.city_id
	GROUP BY 1
	ORDER BY 2 DESC
),

city_rent
AS
(SELECT 
	city_name, 
	estimated_rent
FROM city
)
SELECT 
	cr.city_name,
	cr.estimated_rent,
	ct.total_cx,
	ct.avg_sale_pr_cx,
	ROUND(cr.estimated_rent::numeric/ct.total_cx::numeric, 2) as avg_rent_per_cx
FROM city_rent as cr
JOIN city_table as ct
ON cr.city_name = ct.city_name
ORDER BY 4 DESC


-- Q.9
-- Monthly Sales Growth
-- Sales growth rate: Calculate the percentage growth (or decline) in sales over different time periods (monthly)
-- by each city

WITH
	monthly_sales
AS
(
	SELECT 
		ci.city_name,
		EXTRACT(MONTH FROM sale_date) as month,
		EXTRACT(YEAR FROM sale_date) as YEAR,
		SUM(s.total) as total_sale
	FROM sales as s
	JOIN customers as c
	ON c.customer_id = s.customer_id
	JOIN city as ci
	ON ci.city_id = c.city_id
	GROUP BY 1, 2, 3
	ORDER BY 1, 3, 2
),
growth_ratio
AS
(
		SELECT
			city_name,
			month,
			year,
			total_sale as cr_month_sale,
			LAG(total_sale, 1) OVER(PARTITION BY city_name ORDER BY year, month) as last_month_sale
		FROM monthly_sales
)

SELECT
	city_name,
	month,
	year,
	cr_month_sale,
	last_month_sale,
	ROUND(
		(cr_month_sale-last_month_sale)::numeric/last_month_sale::numeric * 100
		, 2
		) as growth_ratio

FROM growth_ratio
WHERE 
	last_month_sale IS NOT NULL	

-- Q.10
-- Market Potential Analysis
-- Identify top 3 city based on highest sales, return city name, total sale, total rent, total customers, estimated coffee consumer

WITH city_table
AS
(
	SELECT 
		ci.city_name,
		SUM(s.total) as total_revenue,
		COUNT(DISTINCT s.customer_id) as total_cx,
		ROUND(
				SUM(s.total)::numeric/
					COUNT(DISTINCT s.customer_id)::numeric
				,2) as avg_sale_pr_cx
		
	FROM sales as s
	JOIN customers as c
	ON s.customer_id = c.customer_id
	JOIN city as ci
	ON ci.city_id = c.city_id
	GROUP BY 1
	ORDER BY 2 DESC
),

city_rent
AS
(SELECT 
	city_name, 
	estimated_rent,
	Round((population*0.25)/1000000,3) as estimated_coffee_customer_million
FROM city
)
SELECT 
	cr.city_name,
	total_revenue,
	cr.estimated_rent as total_rent,
	ct.total_cx,
	estimated_coffee_customer_million,
	ct.avg_sale_pr_cx,
	ROUND(cr.estimated_rent::numeric/ct.total_cx::numeric, 2) as avg_rent_per_cx
FROM city_rent as cr
JOIN city_table as ct
ON cr.city_name = ct.city_name
ORDER BY 4 DESC


/*
--Recomendation 
City 1: Pune
1. Avg rent per customer is very less,
2. Highest total revenue,
3. avg_sale per customer is also high

City 2: Delhi
1. Highest estimated coffee consumer,
2. Highest total customer which is 68,
3. Acg rent per customer 330 (still under 500)

City 3: Jaipur
1. Highest customernumber which is 69,
2. Avgrent per customer is very less 156,
3. Avg sale per customer is better whichat 11.6k

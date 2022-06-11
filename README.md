These are some sample SQL Queries used to analyze the Sakila DVD rental database.

<b>Q1) Which actor starred in all 16 categories?</b>
```sql
SELECT film_actor.actor_id,actor.first_name,actor.last_name, COUNT(DISTINCT category.category_id) AS tot
FROM film_actor JOIN film USING (film_id)
JOIN actor USING (actor_id)
JOIN film_category USING (film_id)
JOIN category USING (category_id)
GROUP BY film_actor.actor_id,actor.first_name,actor.last_name
HAVING tot = 16
```

<b>Q2) Of those who starred in all 16 categories, which actors starred in the most movies?</b>
```sql
WITH t1 AS (
SELECT film_actor.actor_id,actor.first_name,actor.last_name, COUNT(DISTINCT category.category_id) AS tot
FROM film_actor JOIN film USING (film_id)
JOIN actor USING (actor_id)
JOIN film_category USING (film_id)
JOIN category USING (category_id)
GROUP BY film_actor.actor_id,actor.first_name,actor.last_name
HAVING tot = 16
)
SELECT film_actor.actor_id, actor.first_name, actor.last_name, COUNT(*)
FROM film_actor JOIN actor USING(actor_id)
WHERE film_actor.actor_id IN (SELECT t1.actor_id FROM t1)
GROUP BY film_actor.actor_id, actor.first_name, actor.last_name
ORDER BY COUNT(*) DESC
```

<b>Q3) Get customers who’ve rented movies from all categories</b>
```sql
SELECT customer.customer_id, customer.first_name, customer.last_name, COUNT(DISTINCT category.name) AS tot
FROM rental JOIN inventory USING(inventory_id)
JOIN customer USING(customer_id)
JOIN film USING (film_id)
JOIN film_category USING(film_id)
JOIN category USING(category_id)
GROUP BY customer.customer_id, customer.first_name, customer.last_name
HAVING tot = (SELECT COUNT(DISTINCT category.name) FROM category)
ORDER BY customer.last_name, customer.first_name, customer.customer_id
```

<b>Q4) Get actors who've starred in Sci-Fi movies</b>
```sql
SELECT DISTINCT actor.actor_id, actor.first_name, actor.last_name
FROM actor 
JOIN (
SELECT actor.actor_id, actor.first_name, actor.last_name
FROM film JOIN film_actor USING(film_id)
JOIN actor USING(actor_id)
JOIN film_category USING(film_id)
JOIN category USING(category_id)
WHERE category.name = 'Sci-Fi') t1 ON t1.actor_id = actor.actor_id
```

<b>Q5) Get customers who didn't rent from the top 5 selling actors in rental volume.</b>
```sql
WITH t1 AS (
SELECT a.actor_id, 
COUNT(*) AS rv
FROM actor a
JOIN film_actor fa ON a.actor_id = fa.actor_id
JOIN film f ON fa.film_id = f.film_id
JOIN inventory i ON f.film_id = i.film_id
JOIN rental r ON i.inventory_id = r.inventory_id
GROUP BY a.actor_id
ORDER BY rv DESC
LIMIT 5
)
SELECT *
FROM customer WHERE customer_id NOT IN(
SELECT rental.customer_id
FROM rental JOIN inventory ON rental.inventory_id = inventory.inventory_id
JOIN film ON inventory.film_id = film.film_id
WHERE film.film_id IN (SELECT DISTINCT film_id
FROM film_actor WHERE actor_id IN(SELECT t1.actor_id FROM t1)))
```
<b>Q6) Get each customer's favorite actor using rental volume</b>
```sql
WITH t1 AS (
SELECT customer.customer_id, film_actor.actor_id, COUNT(film_actor.actor_id) AS tot
FROM film_actor JOIN film USING(film_id)
JOIN inventory USING(film_id)
JOIN rental USING(inventory_id)
JOIN customer USING(customer_id)
GROUP BY customer.customer_id, film_actor.actor_id
ORDER BY customer.customer_id
), t2 AS (
SELECT t1.customer_id, MAX(t1.tot) AS tot
FROM t1
GROUP BY t1.customer_id
)
SELECT t1.customer_id, customer.email, t1.actor_id, actor.first_name, actor.last_name,t1.tot
FROM t2 JOIN t1 ON t2.customer_id = t1.customer_id AND t1.tot = t2.tot
JOIN customer ON t1.customer_id = customer.customer_id
JOIN actor ON actor.actor_id = t1.actor_id
```
<b>Q7) Get movies offered in both stores 1 and 2</b>
```sql
SELECT film_id
FROM inventory 
WHERE store_id = 1
INTERSECT
SELECT film_id
FROM inventory
WHERE store_id = 2
```
<b>Q8) For each customer, find the percantage of rentals that were overdue</b>
```sql
WITH t1 AS(SELECT
    r.rental_id AS Rental_ID
		,c.customer_id
    ,c.last_name || ', ' || c.first_name AS Name
    ,c.email
    ,f.title As Film
    ,ROUND(JULIANDAY(r.return_date) - JULIANDAY(r.rental_date), 2) - f.rental_duration AS Overdue_Days
FROM rental AS r
JOIN inventory AS i ON r.inventory_id = i.inventory_id
JOIN film AS f ON i.film_id = f.film_id
JOIN customer AS c ON r.customer_id = c.customer_id
WHERE r.return_date IS NOT NULL
AND Overdue_Days > 0
ORDER BY r.rental_id
),
t2 AS (
SELECT c.customer_id, COUNT(*) AS tot_orders
FROM rental AS r
JOIN inventory AS i ON r.inventory_id = i.inventory_id
JOIN film AS f ON i.film_id = f.film_id
JOIN customer AS c ON r.customer_id = c.customer_id
WHERE r.return_date IS NOT NULL
GROUP BY c.customer_id
),t3 AS(
SELECT t1.customer_id, t1.Name, COUNT(*) AS nb_overdue
FROM t1
GROUP BY t1.customer_id, t1.Name
ORDER BY COUNT(*) DESC
)

SELECT Name, ROUND(1.0*nb_overdue/tot_orders,2) AS Percent_Overdue
FROM t2 JOIN t3 ON t2.customer_id = t3.customer_id
ORDER BY Percent_Overdue DESC
```

<b>Q9) Get actors whose movies were rented the most</b>
```sql
SELECT actor.actor_id, COUNT(*) AS tot_rent
FROM rental JOIN inventory USING(inventory_id)
JOIN film USING(film_id)
JOIN film_actor USING(film_id)
JOIN actor USING(actor_id)
GROUP BY actor.actor_id
ORDER BY tot_rent DESC	
```


<b>Q10) Get the count of customers in each country</b>
```sql
SELECT country.country_id, country.country, COUNT(DISTINCT customer.customer_id) AS tot
FROM rental JOIN customer ON rental.customer_id = customer.customer_id
JOIN address ON address.address_id = customer.address_id
JOIN city ON city.city_id = address.city_id
JOIN country ON country.country_id = city.country_id
GROUP BY country.country_id, country.country
ORDER BY tot DESC
```

<b>Q11) Which films have not been rented yet?</b>
```sql
SELECT *
FROM film
WHERE film_id NOT IN(
SELECT DISTINCT(film.film_id)
FROM rental JOIN inventory USING(inventory_id)
JOIN film USING(film_id)
)
```




/*Query 1 used in SLIDE 1 : Rental Store’s Orders Comparison

finding out how the two stores compare in their count of rental orders during every month for all the years we have data for.
*/


WITH rent_count AS (
			SELECT DISTINCT DATE_PART('year', rt.rental_date) AS year, DATE_PART('month', rt.rental_date) AS month, sf.store_id,
					COUNT(DATE_PART('month', rt.rental_date)) OVER (PARTITION BY sf.store_id ORDER BY DATE_PART('month', rt.rental_date)) AS count
			FROM staff sf
			JOIN store st
			ON sf.store_id = st.store_id
			JOIN rental rt
			ON sf.staff_id = rt.staff_id)

SELECT rc.year, 
		CASE
			WHEN rc.month = 1 THEN 'Jan'
			WHEN rc.month = 2 THEN 'Feb'
			WHEN rc.month = 3 THEN 'Mar'
			WHEN rc.month = 4 THEN 'Apr'
			WHEN rc.month = 5 THEN 'May'
			WHEN rc.month = 6 THEN 'Jun'
			WHEN rc.month = 7 THEN 'Jul'
			WHEN rc.month = 8 THEN 'Aug'
			WHEN rc.month = 9 THEN 'Sep'
			WHEN rc.month = 10 THEN 'Oct'
			WHEN rc.month = 11 THEN 'Nov'
			WHEN rc.month = 12 THEN 'Dec'
		END AS month,
	rc.store_id,
	rc.count
FROM rent_count rc;




/*Query 2 used in SLIDE 2 : Payment Insight for Top 10 paying customers 

For each of the top 10 paying customers, I would like to find out the difference across their monthly payments during 2007. 
query to compare the payment amounts in each successive month.*/


WITH topC AS (	
		SELECT DISTINCT py.customer_id cx_id, CONCAT(cs.first_name, ' ', cs.last_name) fullname, DATE_TRUNC('month', py.payment_date) pay_month,
		COUNT(DATE_TRUNC('month', py.payment_date)) OVER (PARTITION BY py.customer_id ORDER BY (DATE_TRUNC('month', py.payment_date))) payment_count,
		SUM(py.amount) OVER (PARTITION BY py.customer_id ORDER BY (DATE_TRUNC('month', py.payment_date))) total_amt_paid
		FROM payment py
		JOIN customer cs
		ON py.customer_id = cs.customer_id
		WHERE py.customer_id IN (SELECT top10.cx_id
					FROM(
						SELECT py.customer_id cx_id,
							SUM(py.amount)
						FROM payment py
						JOIN customer cs
						ON py.customer_id = cs.customer_id
						GROUP BY 1
						ORDER BY 2 DESC
						LIMIT 10) AS top10)
		ORDER BY 1)
		
SELECT topC.fullname, topC.pay_month, topC.total_amt_paid,
	LEAD(topC.total_amt_paid) OVER (PARTITION BY topC.cx_id ORDER BY topC.fullname),
	COALESCE((LEAD(topC.total_amt_paid) OVER (PARTITION BY topC.cx_id ORDER BY topC.fullname) - topC.total_amt_paid ), 0) monthly_diff
FROM topC
ORDER BY 5 DESC;




/*Query 3 used in SLIDE 3 : Rental Count for each DVD

Insight into the movies that families are watching. 
The following categories are considered family movies: Animation, Children, Classics, Comedy, Family and Music.
*/



WITH filmCat AS (
                SELECT fc.film_id film_id, fc.category_id category_id, c.name AS name
                FROM film_category fc
                    JOIN category c
                    ON fc.category_id = c.category_id
                WHERE c.name IN ('Animation', 'Children', 'Classics', 'Comedy', 'Family', 'Music')),

    rentDate AS (
                SELECT i.film_id film_id, r.rental_date rental_date
                FROM rental r
                    JOIN inventory i
                    ON r.inventory_id = i.inventory_id)

SELECT DISTINCT f.title film_title, fc.name AS category_name, COUNT(rd.rental_date) OVER (PARTITION BY f.title ORDER BY fc.name ASC) AS rental_count
FROM film f
    JOIN filmCat fc
    ON f.film_id = fc.film_id
    JOIN rentDate rd
    ON f.film_id = rd.film_id
ORDER BY 2,1;





/*Query 4 used in SLIDE 4 : Rental duration distribution in categories

Four quartiles insight into the family-friendly film category,
 and the corresponding count of movies within each combination of film category for each corresponding rental duration category. 
*/


WITH filmCat AS (
                SELECT fc.film_id film_id, fc.category_id category_id, c.name AS name
                FROM film_category fc
                    JOIN category c
                    ON fc.category_id = c.category_id
                WHERE c.name IN ('Animation', 'Children', 'Classics', 'Comedy', 'Family', 'Music')),
                 
    quart AS (
                SELECT fc.name AS name,  NTILE(4) OVER (ORDER BY f.rental_duration) AS standard_quartile 
                FROM film f
                    JOIN filmCat fc
                    ON f.film_id = fc.film_id)


SELECT DISTINCT q.name AS name, q.standard_quartile, COUNT(q.standard_quartile) OVER (PARTITION BY q.standard_quartile ORDER BY q.name ASC) AS count
FROM quart q
ORDER BY 1, 2;
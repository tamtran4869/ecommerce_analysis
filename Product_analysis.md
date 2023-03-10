# Product analysis
It helps to optimise products lines, prepare stocks and specify marketing strategy or website design
for each product.

Note: There are some complex queries (e.g. Q14, Q15) which could be presented in the form of cte, subquery or temporary table, depending on different purposes. This project prefers cte and subquery to combine all into 1 query, which is more straightforward for running through Python.

## Question 12

Pull monthly trends to date for number of sales, total revenue, and total margin generated
for the business.

_Received date: Jan 04, 2013_

#### SQL query 
```sql
SELECT
	YEAR(created_at) AS year,
	MONTH(created_at) AS month,
	COUNT(DISTINCT order_id) AS number_of_sales,
	SUM(price_usd) AS total_revenue,
	SUM(price_usd-cogs_usd) AS total_profit
FROM orders
WHERE 
	created_at < '2013-01-04'
GROUP BY
	YEAR(created_at),
	MONTH(created_at);
```
#### Command & Results

```
python3 connect.py --question 12 --db 'mavenfuzzyfactory user password'
```

![image](https://user-images.githubusercontent.com/114192113/211810117-7a31e2a9-6f37-40e3-9299-d6beb001d52f.png)

![image](https://user-images.githubusercontent.com/114192113/211812335-ac620bde-0b2f-4115-881a-eba3fede6c45.png)

#### Comments

The Mr Fuzzy product had a steady increase in orders which brought rises in revenue and profit as well.
The orders increased surprisingly in November and were back to the normal speed in December.

Note: the company used this data as a baseline to test new product launching.

## Question 13

The company introduced a new product - Love bear in Jan 2013.
Pull monthly order volume, overall conversion rates, revenue per session, and a 
breakdown of sales by product, all for the time period since April 1, 2012 to compare the 2 products.

_Received date: Apr 05, 2013_

#### SQL query 
```sql
SELECT
	YEAR(ws.created_at) AS year,
	MONTH(ws.created_at) AS month,
	COUNT(DISTINCT ws.website_session_id) AS sessions,
	COUNT(DISTINCT o.order_id) AS orders,
	COUNT(DISTINCT o.order_id)/COUNT(DISTINCT ws.website_session_id) AS conv_rate,
	SUM(o.price_usd)/COUNT(DISTINCT ws.website_session_id) AS revenue_per_session,
	COUNT(DISTINCT CASE WHEN o.primary_product_id=1 THEN o.order_id ELSE NULL END) AS product_1_orders,
	COUNT(DISTINCT CASE WHEN o.primary_product_id=2 THEN o.order_id ELSE NULL END) AS product_2_orders
FROM website_sessions ws
LEFT JOIN orders o
	ON ws.website_session_id = o.website_session_id
WHERE 
	ws.created_at < '2013-04-05' 
	AND ws.created_at >'2012-04-01'
GROUP BY 1,2;
```
#### Command & Results

```
python3 connect.py --question 13 --db 'mavenfuzzyfactory user password'
```

![image](https://user-images.githubusercontent.com/114192113/211821591-f4d96e93-7d69-446a-9dbe-d887638e84e4.png)

![image](https://user-images.githubusercontent.com/114192113/211849875-00c2c27d-ff02-426d-96b1-113a5a5c5ffe.png)

#### Comments
After the holiday, the orders reduced and fluctuated.
Since the new product launched in Jan 2013, the Mr Fuzzy orders decreased and stayed stable, while 
the Love Bear orders were unstable. 

However, the conversion rates kept going up. It could be the general improvements of business (e.g. due to advances in marketing, website) or the excellent performance of the new product. 

Note: the complete data for the last month had yet to be collected.

## Question 14

Pull CTR from /products since the new product launch on 2013-01-06, by product,
and compare to the 3 months leading up to launch as a baseline.

_Received date: Apr 06, 2013_

#### SQL query 
There are some steps to extract the data:
1. Get sessions viewing the /product page and classify it: before and after product 2 launching. 

![image](https://user-images.githubusercontent.com/114192113/211909711-6ab085f8-d943-474e-bb46-779b6c4d1c57.png)

2. Get the next pageview id of these sessions.

![image](https://user-images.githubusercontent.com/114192113/211909854-fa02dc51-9f18-4093-b733-8444428dac1e.png)

3. Get url of the next pageviews by the next pageview id.

![image](https://user-images.githubusercontent.com/114192113/211909995-b9ff76f6-8eb6-4893-8b99-217d5172d9f7.png)

4. Compute the count sessions grouping by the time period

![image](https://user-images.githubusercontent.com/114192113/211912426-f9ab7aa9-fff9-4d46-9e4e-d3165ab2e413.png)

5. Calculate CTR of /products to next pageview, CTR of /products to /the-original-mr-fuzzy and CTR of /products to the /the-forever-love-bear

![image](https://user-images.githubusercontent.com/114192113/211910705-bd7105ea-183c-42d6-b091-7266d02d4d3f.png)

The full SQL query:

```sql
-- Create cte to get next pageview id from sessions viewing /products.
WITH next_pageview_id AS (
SELECT 
	pp.website_session_id,
	pp.time_period,
	MIN(wp.website_pageview_id) AS next_pageview
FROM 	(
	SELECT -- Get sessions viewing products
		website_session_id,
		website_pageview_id,
		created_at,
		CASE
			WHEN created_at < '2013-01-06' THEN 'A. Before_Product2'
			WHEN created_at >= '2013-01-06' THEN 'B. After_Product2'
			ELSE NULL
		END AS time_period
	FROM website_pageviews
	WHERE 
		created_at < '2013-04-06' AND
		created_at > '2012-10-06' AND
		pageview_url = '/products'
	) AS pp
LEFT JOIN website_pageviews wp
	ON wp.website_session_id  = pp.website_session_id
	AND wp.website_pageview_id > pp.website_pageview_id
GROUP BY 1,2)


-- Compute session counts and CTR 
SELECT 
	time_period,
	sessions,
	next_pageview_sessions,
	next_pageview_sessions/sessions AS next_page_ctr,
	to_mrfuzzy,
	to_mrfuzzy/sessions AS mrfuzzy_ctr,
	to_lovebear,
	to_lovebear/sessions AS lovebear_ctr
FROM 	(
	SELECT -- Count sessions
		time_period,
		COUNT(DISTINCT website_session_id) AS sessions,
		COUNT(DISTINCT CASE WHEN next_pageview_url IS NOT NULL THEN website_session_id ELSE NULL END) AS next_pageview_sessions,
		COUNT(DISTINCT CASE WHEN next_pageview_url = '/the-original-mr-fuzzy' THEN website_session_id ELSE NULL END) AS to_mrfuzzy,
		COUNT(DISTINCT CASE WHEN next_pageview_url = '/the-forever-love-bear' THEN website_session_id ELSE NULL END) AS to_lovebear
	FROM	(	
		SELECT -- Get next pageview url
			npi.website_session_id,
			npi.time_period,
			wp.pageview_url AS next_pageview_url
		FROM next_pageview_id AS npi
		LEFT JOIN website_pageviews wp 
			ON wp.website_pageview_id = npi.next_pageview
		) AS next_pageview_url
	GROUP BY 1
	) AS count_sessions;

```
#### Command & Results

```
python3 connect.py --question 13 --db 'mavenfuzzyfactory user password'
```

#### Comments
The percentage of sessions clicking to the next page after seeing the product listing page after launching the love bear product was higher than before. The various choice of products could explain it. So, adding more products helps improve CTR.

While the percentage of clicks to the Mr Fuzzy product decreased after adding the love bear, it still accounted for significant amounts. Assuming using the same marketing strategy, one possible reason is that the Mr Fuzzy could be more suitable for a mass target audience than the Love Bear (e.g. only for couples). 

## Question 15

Analyse the conversion funnels for each product (from product page to thankyou page).

_Received date: Apr 10, 2013_

#### SQL query 
There are some steps for this request.
1. Filter to keep only sessions from the product page (Mr Fuzzy or Love Bear, cart, shipping, billing, and thank you page)
![image](https://user-images.githubusercontent.com/114192113/211917122-51349451-950d-439a-afbf-f99912060483.png)

2. Mark the pageview url into 1 and 0 (or spread the pageview_url into 1 and 0)
![image](https://user-images.githubusercontent.com/114192113/211917452-f170cb4b-5a79-4bb3-8093-572b2704472f.png)

3. Flag the path of each session (e.g the session 63513 went to the thankyou page)

![image](https://user-images.githubusercontent.com/114192113/211917917-cdbf8d78-3aa7-45f0-b994-dbb5ca2068d1.png)

4. Group by product pages, count session and compile CTR of each step.

![image](https://user-images.githubusercontent.com/114192113/211918119-2fe09544-eba4-4da7-a932-a75948ebf1a3.png)

The fill query:
```sql
-- Flag the path of each session
WITH funnel_flag AS (
SELECT
	website_session_id,
	product_page_seen,
	MAX(cart_page) AS to_cart,
	MAX(shipping_page) AS to_shipping,
	MAX(billing_page) AS to_billing,
	MAX(thankyou_page) AS to_thankyou
FROM(	
	SELECT -- Spread the pageview_url into 1 or 0
		ssp.website_session_id,
		ssp.product_page_seen,
		CASE WHEN wp.pageview_url = '/cart' THEN 1 ELSE 0 END AS cart_page,
		CASE WHEN wp.pageview_url = '/shipping' THEN 1 ELSE 0 END AS shipping_page,
		CASE WHEN wp.pageview_url = '/billing-2' THEN 1 ELSE 0 END AS billing_page,
		CASE WHEN wp.pageview_url = '/thank-you-for-your-order' THEN 1 ELSE 0 END AS thankyou_page
	FROM ( SELECT -- Keep only sessions from products page
			website_session_id,
			website_pageview_id,
			pageview_url AS product_page_seen
		FROM website_pageviews
		WHERE 
			created_at < '2013-04-10'
			AND created_at > '2013-01-06'
			AND pageview_url IN ('/the-original-mr-fuzzy','/the-forever-love-bear')
	) AS ssp
	LEFT JOIN website_pageviews wp
		ON wp.website_session_id = ssp.website_session_id
		AND wp.website_pageview_id > ssp.website_pageview_id
	) AS page_flag
GROUP BY 1,2)

-- Compute metrics
SELECT *,
	to_cart/sessions AS product_conv_rate,
	to_shipping/to_cart AS cart_conv_rate,
	to_billing/to_shipping AS shipping_conv_rate,
	to_thankyou/to_billing AS billing_conv_rate
FROM (
	SELECT -- Count sessions
		product_page_seen,
		COUNT(DISTINCT website_session_id) AS sessions,
		COUNT(DISTINCT CASE WHEN to_cart = 1 THEN website_session_id ELSE NULL END) AS to_cart,
		COUNT(DISTINCT CASE WHEN to_shipping = 1 THEN website_session_id ELSE NULL END) AS to_shipping,
		COUNT(DISTINCT CASE WHEN to_billing = 1 THEN website_session_id ELSE NULL END) AS to_billing,
		COUNT(DISTINCT CASE WHEN to_thankyou = 1 THEN website_session_id ELSE NULL END) AS to_thankyou
	FROM funnel_flag
	GROUP BY 1
	) AS funnel_sessions;
```
#### Command & Results

```
python3 connect.py --question 15 --db 'mavenfuzzyfactory user password'
```
![image](https://user-images.githubusercontent.com/114192113/211920466-93ce2c74-33d0-4c31-b2a3-bef40039bf2c.png)

#### Comments

There was not much difference between the products. Only one significant difference was in the step from view product to add cart. The new product gives a higher CTR in this step. 

Assuming the website designs were the same, it showed that the new product more satisfied the demand of users than Mr Fuzzy, and it could be explained by the specification of the Love Bear focusing on couples compared to the general audiences of Mr Fuzzy.

## Question 16

Compare the month before vs the month after offering a function to add 2nd product into the cart with CTR from the /cart page, Avg Products per Order, Average Order Value (AOV), and overall revenue per /cart page view.

_Received date: Nov 22, 2013_

#### SQL query 

First, create 3 tables:
1. Filter data to get only sessions made to the /cart page (session_seeing_cart).

![image](https://user-images.githubusercontent.com/114192113/211937495-82f24103-839f-4e85-ae1d-5604968991ab.png)

2. From table 1, get the next pageview id of the cart sessions (session_viewing_more).

![image](https://user-images.githubusercontent.com/114192113/211938348-fe28d2fb-88c5-4793-866e-5001b03f9234.png)

3. From table 1, filter and keep only cart sessions placed order and get data related to the orders (session_place_order).

![image](https://user-images.githubusercontent.com/114192113/211938562-8235819b-73db-4b59-818e-103fe7fc1bdb.png)

Join 3 tables into a full table and mark which cart sessions place order, which click to next pages, then compile required metrics.

The full query:
```sql
-- Table 1
WITH session_seeing_cart AS(
SELECT 
	CASE
		WHEN created_at < '2013-09-25' THEN 'A. Before_Cross_Sell'
		WHEN created_at >= '2013-09-25' THEN 'B. After_Cross_Sell'
		ELSE NULL
	END AS time_period,
	website_session_id AS cart_session_id,
	website_pageview_id AS cart_pageview_id
FROM website_pageviews
WHERE 
	created_at BETWEEN '2013-08-25' AND '2013-10-25'
	AND pageview_url = '/cart'),

--Table 2	
session_viewing_more AS(
SELECT 
	ssc.time_period,
	ssc.cart_session_id,
	MIN(wp.website_pageview_id) AS next_pageview_id
FROM session_seeing_cart ssc
LEFT JOIN website_pageviews wp
	ON wp.website_session_id = ssc.cart_session_id
	AND wp.website_pageview_id > ssc.cart_pageview_id
GROUP BY 
	1,2
HAVING 
	MIN(wp.website_pageview_id) IS NOT NULL),

--Table 3
session_place_order AS (
SELECT
	time_period,
	cart_session_id,
	order_id,
	items_purchased,
	price_usd
FROM session_seeing_cart ssc
INNER JOIN orders o
	ON ssc.cart_session_id = o.website_session_id)

-- Join 3 tables and compute metrics	
SELECT
	time_period,
	COUNT(DISTINCT cart_session_id) AS cart_sessions,
	SUM(click_next_page) AS clickthroughs,
	SUM(click_next_page)/COUNT(DISTINCT cart_session_id) AS cart_ctr,
	SUM(items_purchased)/SUM(place_order) AS product_per_order,
	SUM(price_usd)/SUM(place_order) AS average_order_rev,
	SUM(price_usd)/COUNT(DISTINCT cart_session_id) AS rev_per_cart_session
FROM	(
	SELECT  -- Mark sessions click to next page and sessions place order
		ssc.time_period,
		ssc.cart_session_id,
		CASE WHEN svm.cart_session_id IS NULL THEN 0 ELSE 1 END AS click_next_page,
		CASE WHEN spo.order_id IS NULL THEN 0 ELSE 1 END AS place_order,
		spo.items_purchased,
		spo.price_usd
	FROM session_seeing_cart ssc
	LEFT JOIN session_viewing_more svm
		ON ssc.cart_session_id = svm.cart_session_id
	LEFT JOIN session_place_order spo
		ON ssc.cart_session_id = spo.cart_session_id
	) AS full
GROUP BY time_period;
```
#### Command & Results

```
python3 connect.py --question 16 --db 'mavenfuzzyfactory user password'
```

![image](https://user-images.githubusercontent.com/114192113/211939070-d1bf0098-40bd-4ec1-98f4-6e56354e19fc.png)

#### Comments

The data showed a slight improvement in all metrics since adding new functions.

## Question 17

Please pull monthly product refund rates, by product, and confirm the quality issues fixed (Sep 2014).

_Received date: Oct 15, 2014_

#### SQL query 
The first step is to count order_id and order_id_refund by products, then compile the rates.

```sql
-- Compile refund rates
SELECT
	year,
	month,
	order_p1,
	refund_p1/order_p1 AS refund_p1_rate,
	order_p2,
	refund_p2/order_p2 AS refund_p2_rate,
	order_p3,
	refund_p3/order_p3 AS refund_p3_rate,
	order_p4,
	refund_p4/order_p4 AS refund_p4_rate
FROM (
	SELECT -- Count orders
		YEAR(oi.created_at) AS year,
		MONTH(oi.created_at) AS month,
		COUNT(DISTINCT CASE WHEN oi.product_id =1 THEN oi.order_item_id ELSE NULL END) AS order_p1,
		COUNT(DISTINCT CASE WHEN oi.product_id =1 THEN oir.order_item_id ELSE NULL END) AS refund_p1,
		COUNT(DISTINCT CASE WHEN oi.product_id =2 THEN oi.order_item_id ELSE NULL END) AS order_p2,
		COUNT(DISTINCT CASE WHEN oi.product_id =2 THEN oir.order_item_id ELSE NULL END) AS refund_p2,
		COUNT(DISTINCT CASE WHEN oi.product_id =3 THEN oi.order_item_id ELSE NULL END) AS order_p3,
		COUNT(DISTINCT CASE WHEN oi.product_id =4 THEN oir.order_item_id ELSE NULL END) AS refund_p3,
		COUNT(DISTINCT CASE WHEN oi.product_id =4 THEN oi.order_item_id ELSE NULL END) AS order_p4,
		COUNT(DISTINCT CASE WHEN oi.product_id =4 THEN oir.order_item_id ELSE NULL END) AS refund_p4
	FROM order_items oi
	LEFT JOIN order_item_refunds oir
		ON oi.order_item_id = oir.order_item_id
	WHERE oi.created_at < '2014-10-15'
	GROUP BY 1,2
	) AS count;
```
#### Command & Results

```
python3 connect.py --question 17 --db 'mavenfuzzyfactory user password'
```

![image](https://user-images.githubusercontent.com/114192113/212083934-6fb61b92-187c-4155-9dc7-8033c75ce3b0.png)

![image](https://user-images.githubusercontent.com/114192113/212083970-4817230e-93ce-4a52-944a-4df8ea91f937.png)

#### Comments

The second chart showed that the refund rate of product 1 - Mr Fuzzy reduced significantly in the latest month after a soar in Aug and Sep 2014. The problem was fixed.

Besides, some interesting observations were shown in the charts. Product 1 and product 2 had some struggles with refunds in the beginning. While product 1 needed around 20 months to stabilise the quality of the product, product 2 spent around 7 months to do this. The number of orders did not have a positive relationship with the refund rates.

# olist-e-commerce-SQL
CREATE TABLE Olist_customers_dataset(
customer_id	VARCHAR,
customer_unique_id	VARCHAR,
customer_zip_code_prefix INT,	
customer_city VARCHAR,
customer_state VARCHAR
)

CREATE TABLE olist_geolocation_dataset(
geolocation_zip_code_prefix INT,	
geolocation_lat	FLOAT,
geolocation_lng	FLOAT,
geolocation_city VARCHAR,	
geolocation_state CHAR (2)
);

CREATE TABLE olist_order_items_dataset(
order_id VARCHAR,
order_item_id INT,
product_id	VARCHAR,
seller_id	VARCHAR,
shipping_limit_date	DATE,
price FLOAT,
freight_value FLOAT
)

CREATE TABLE olist_order_payments_dataset(
order_id VARCHAR,	
payment_sequential	INT,
payment_type VARCHAR,	
payment_installments INT,
payment_value FLOAT
)


CREATE TABLE olist_order_reviews_dataset(
review_id VARCHAR,
order_id VARCHAR,
review_score INT,	
review_comment_title TEXT,	
review_comment_message TEXT,
review_creation_date TIMESTAMP,
review_answer_timestamp TIMESTAMP
);

CREATE TABLE olist_orders_dataset(
order_id VARCHAR,	
customer_id	VARCHAR,
order_status VARCHAR,	
order_purchase_timestamp TIMESTAMP,	
order_approved_at TIMESTAMP,	
order_delivered_carrier_date TIMESTAMP, 
order_delivered_customer_date TIMESTAMP, 
order_estimated_delivery_date TIMESTAMP
);
SELECT COUNT (*) FROM olist_orders_dataset;
SELECT table_schema, COUNT(*)
FROM information_schema.tables
WHERE table_name = 'olist_orders_dataset'
GROUP BY table_schema

SELECT table_schema,table_name
FROM information_schema.tables
WHERE table_name LIKE '%olist_orders%';

SHOW default_transaction_read_only;



CREATE TABLE olist_products_dataset(
product_id VARCHAR,
product_category_name VARCHAR,	
product_name_lenght	INT,
product_description_lenght INT,
product_photos_qty	INT,
product_weight_g	INT,
product_length_cm	INT,
product_height_cm	INT,
product_width_cm INT
);

CREATE TABLE olist_sellers_dataset(
seller_id	VARCHAR,
seller_zip_code_prefix	INT,
seller_city	VARCHAR,
seller_state CHAR(3)
);


CREATE TABLE olist_category_name_translation_dataset(
product_category_name	VARCHAR,
product_category_name_english VARCHAR
);

SELECT * FROM olist_category_name_translation_dataset
SELECT * FROM olist_sellers_dataset;
SELECT * FROM olist_products_dataset;
SELECT * FROM olist_order_reviews_dataset;
SELECT * FROM olist_order_payments_dataset
SELECT * FROM olist_order_items_dataset
SELECT * FROM olist_customers_dataset;
SELECT * FROM olist_geolocation_dataset;
SELECT * FROM olist_orders_dataset;

--- SALES AND REVENUE PERFORMANCE ---
/* how has total revenue changed over time */
SELECT TO_CHAR(O.order_purchase_timestamp,'Month') AS month,
SUM(OI.price) AS total_revenue 
FROM olist_orders_dataset O
JOIN olist_order_items_dataset OI
ON O.order_id = OI.order_id
GROUP BY month
ORDER BY total_revenue DESC

/* which product categories generated the highest revenue */
SELECT 
P.product_category_name,
ROUND(SUM(OI.price)::numeric,2) AS total_revenue
FROM olist_order_items_dataset OI
JOIN olist_products_dataset P
ON OI.product_id = P.product_id
GROUP BY P.product_category_name
ORDER BY total_revenue DESC
LIMIT 10

/* what is the average order value (AOV) by poduct category */
SELECT 
 P.product_category_name,
  ROUND(SUM(OI.price)::numeric/ NULLIF(COUNT(DISTINCT O.order_id),0)::numeric,2) AS avg_order_value
  FROM olist_orders_dataset O
  JOIN olist_order_items_dataset OI
  ON O.order_id = OI.order_id
  JOIN olist_products_dataset P
  ON P.product_id = OI.product_id
  GROUP BY P.product_category_name
 ORDER BY avg_order_value DESC
 LIMIT 10
/* which sellers contribute the most to total sales */
SELECT
    s.seller_id,
    SUM(oi.price) AS seller_revenue
FROM olist_order_items_dataset oi
JOIN olist_sellers_dataset s 
ON oi.seller_id = s.seller_id
GROUP BY s.seller_id
ORDER BY seller_revenue DESC
LIMIT 10
--- CUSTOMER BEHAVIOR AND SEGMENTATION ---
/* how many unique customers place repeat orders */
SELECT
    CASE 
        WHEN COUNT(o.order_id) > 1 THEN 'Repeat Customer'
        ELSE 'One-time Customer'
    END AS customer_type,
    COUNT(DISTINCT c.customer_id) AS customer_count
FROM olist_customers_dataset c
JOIN olist_orders_dataset o 
ON c.customer_id = o.customer_id

/* what is the average number of orders per customers */
SELECT
    ROUND(COUNT(order_id)::decimal / COUNT(DISTINCT customer_id), 2) 
    AS avg_orders_per_customer
FROM olist_orders_dataset;
/* which states have the highest number of customers */
SELECT
    customer_state,
    COUNT(DISTINCT customer_id) AS total_customers
FROM olist_customers_dataset
GROUP BY customer_state
ORDER BY total_customers DESC;
/* what is customer spending by location */
SELECT
    c.customer_state,
    SUM(oi.price) AS total_spending
FROM olist_customers_dataset c
JOIN olist_orders_dataset o 
ON c.customer_id = o.customer_id
JOIN olist_order_items_dataset oi 
ON o.order_id = oi.order_id
GROUP BY c.customer_state
ORDER BY total_spending DESC
LIMIT 10
--- ORDER FULFILLMENT AND DELIVERY ---
/* what is the average delivery time */
SELECT
    AVG(order_delivered_customer_date - order_purchase_timestamp)
    AS avg_delivery_days
FROM olist_orders_dataset
WHERE order_delivered_customer_date IS NOT NULL;
/* which states experience the longest delivery delays */
SELECT
    c.customer_state,
   AVG(o.order_delivered_customer_date - o.order_purchase_timestamp)
    AS avg_delivery_days
FROM olist_orders_dataset o
JOIN olist_customers_dataset c 
ON o.customer_id = c.customer_id
WHERE o.order_delivered_customer_date IS NOT NULL
GROUP BY c.customer_state
ORDER BY avg_delivery_days DESC;
/* how often are orders delivered late */
 SELECT
    COUNT(*) AS late_orders
FROM olist_orders_dataset
WHERE order_delivered_customer_date > order_estimated_delivery_date;
/* does delivery time affect customer review scores */
SELECT
    r.review_score,
    AVG(o.order_delivered_customer_date - o.order_purchase_timestamp)
    AS avg_delivery_days
FROM olist_orders_dataset o
JOIN olist_order_reviews_dataset r 
ON o.order_id = r.order_id
WHERE o.order_delivered_customer_date IS NOT NULL
GROUP BY r.review_score
ORDER BY r.review_score;
--- CUSTOMER SATISFACTION AND REVIEWS ---
/* what is the average review score by product category */
SELECT
    p.product_category_name,
    ROUND(AVG(r.review_score)::numeric,2) AS avg_review_score
FROM olist_order_reviews_dataset r
JOIN olist_orders_dataset o 
ON r.order_id = o.order_id
JOIN olist_order_items_dataset oi 
ON o.order_id = oi.order_id
JOIN olist_products_dataset p 
ON oi.product_id = p.product_id
GROUP BY p.product_category_name
ORDER BY avg_review_score DESC;
/* which sellers receive the highest and lowest ratings */
SELECT
    oi.seller_id,
    ROUND(AVG(r.review_score)::numeric,2) AS avg_rating
FROM olist_order_items_dataset oi
JOIN olist_order_reviews_dataset r 
ON oi.order_id = r.order_id
GROUP BY oi.seller_id
ORDER BY avg_rating DESC;
/* do late deliveries lead to poor reviews */
SELECT
    CASE
        WHEN o.order_delivered_customer_date > o.order_estimated_delivery_date
        THEN 'Late'
        ELSE 'On-time'
    END AS delivery_status,
    ROUND(AVG(r.review_score)::numeric,2) AS avg_review_score
FROM olist_orders_dataset o
JOIN olist_order_reviews_dataset r 
ON o.order_id = r.order_id
GROUP BY delivery_status;

--- PRODUCT PERFORMANCE AND PRICING ---
/* which products are most frequently ordered */
SELECT
    product_id,
    COUNT(*) AS order_count
FROM olist_order_items_dataset
GROUP BY product_id
ORDER BY order_count DESC
LIMIT 10;
/* which categories have the highest average price */
SELECT
    p.product_category_name,
    ROUND(AVG(oi.price)::numeric,2) AS avg_price
FROM olist_order_items_dataset oi
JOIN olist_products_dataset p 
ON oi.product_id = p.product_id
GROUP BY p.product_category_name
ORDER BY avg_price DESC
LIMIT 10
/* is there a relationship between product price and review score */
SELECT
    ROUND(AVG(oi.price)::numeric,2) AS avg_price,
    ROUND(AVG(r.review_score)::numeric,2) AS avg_review_score
FROM olist_order_items_dataset oi
JOIN olist_order_reviews_dataset r 
ON oi.order_id = r.order_id;
/* which categories have high sales but low ratings */
SELECT
    p.product_category_name,
    ROUND(SUM(oi.price)::numeric,2) AS total_sales,
    ROUND(AVG(r.review_score)::numeric,2) AS avg_rating
FROM olist_order_items_dataset oi
JOIN olist_products_dataset p 
ON oi.product_id = p.product_id
JOIN olist_order_reviews_dataset r 
ON oi.order_id = r.order_id
GROUP BY p.product_category_name
HAVING AVG(r.review_score) < 3
ORDER BY total_sales DESC;

--- MARKETING AND BUSINESS INSIGHTS ---
/* what payment methods are most comonly used */
SELECT
    payment_type,
    COUNT(*) AS total_payments
FROM olist_order_payments_dataset
GROUP BY payment_type
ORDER BY total_payments DESC;
/* does payment type affect order value */
SELECT
    payment_type,
    ROUND(AVG(payment_value)) AS avg_payment_value
FROM olist_order_payments_dataset
GROUP BY payment_type
ORDER BY avg_payment_value DESC
/* what is the revenue contribution by payment type */
 SELECT
    payment_type,
    ROUND(SUM(payment_value)::numeric,2) AS total_revenue
FROM olist_order_payments_dataset
GROUP BY payment_type
ORDER BY total_revenue DESC;
/* are installment payments linked to higher order values */
 SELECT
    payment_installments,
    ROUND(AVG(payment_value)::numeric,2) AS avg_order_value
FROM olist_order_payments_dataset
GROUP BY payment_installments
ORDER BY payment_installments
LIMIT 10;

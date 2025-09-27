# PL/SQL Window Functions – Coffee Retail Company
This repository contains the PL/SQL Window Functions Mastery Project for the course Database Development with PL/SQL. It implements a complete analysis of sales data for a coffee retail company with branches in different regions.

## Author
**Name:** CYIZERE SIBORUREMA Elie  
**ID:** 27492  
**N.B.:** [Click here to view each step performed from this PL/SQL Window Function for our Coffee Retail Company](https://github.com/emerick149/plsql-window-function-CYIZERE-SIBORUREMA-Elie/tree/main/word%20document)

here you will find the word document containg each step performed  with scripts and screenshoot,ER Diagrams and  their outcome and interpretation for each step asked 

---

## 1. Business Problem  
We are a **coffee retail company** operating multiple branches across different regions.  
The Sales & Marketing department manages customer relationships, product portfolios, and regional sales performance.

### Data Challenge  
Management wants to understand which coffee products perform best in each region and how customer buying behaviour evolves over time. Currently, sales data is collected from all branches but not analyzed for ranking products, tracking growth, or segmenting customers. This makes it difficult to plan marketing campaigns and inventory allocation.

### Expected Outcome  
Generate insights to identify the top-selling products per region, segment customers by spending levels, and measure month-over-month sales growth to support better marketing decisions and stock planning.

---

## 2. Success Criteria  

| Goal | Window Function |
|------|-----------------|
| Top 5 products per region/quarter | `RANK()` |
| Running monthly sales totals | `SUM() OVER()` |
| Month-over-month growth | `LAG()` / `LEAD()` |
| Customer quartiles | `NTILE(4)` |
| 3-month moving averages | `AVG() OVER()` |

---

## 3. Database Schema  

We designed a relational database with three related tables:

- **customers**: keeps customer information (`customer_id`, name, region).  
- **products**: keeps product catalogue (`product_id`, name, category).  
- **transactions**: records sales (`transaction_id`, `customer_id` (FK), `product_id` (FK), `sale_date`, amount).  

Primary keys uniquely identify each record, and foreign keys enforce referential integrity.  
Oracle **sequences** automatically generate IDs for each table.

### SQL Scripts

```sql
-- Create tables
CREATE TABLE customers (
  customer_id NUMBER PRIMARY KEY,
  name        VARCHAR2(100) NOT NULL,
  region      VARCHAR2(50)  NOT NULL
);

CREATE TABLE products (
  product_id NUMBER PRIMARY KEY,
  name       VARCHAR2(100) NOT NULL,
  category   VARCHAR2(50)  NOT NULL
);

CREATE TABLE transactions (
  transaction_id NUMBER PRIMARY KEY,
  customer_id    NUMBER NOT NULL,
  product_id     NUMBER NOT NULL,
  sale_date      DATE   NOT NULL,
  amount         NUMBER(12,2) NOT NULL,
  CONSTRAINT fk_customer FOREIGN KEY (customer_id)
    REFERENCES customers(customer_id),
  CONSTRAINT fk_product FOREIGN KEY (product_id)
    REFERENCES products(product_id)
);

-- Create sequences
CREATE SEQUENCE customers_seq START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE products_seq START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE transactions_seq START WITH 1 INCREMENT BY 1;

-- Insert sample data using sequences
INSERT INTO customers (customer_id, name, region)
VALUES (customers_seq.NEXTVAL, 'John Doe', 'Kigali');

INSERT INTO customers (customer_id, name, region)
VALUES (customers_seq.NEXTVAL, 'Jane Smith', 'Musanze');

INSERT INTO products (product_id, name, category)
VALUES (products_seq.NEXTVAL, 'Coffee Beans', 'Beverages');

INSERT INTO products (product_id, name, category)
VALUES (products_seq.NEXTVAL, 'Espresso Machine', 'Equipment');

INSERT INTO transactions (transaction_id, customer_id, product_id, sale_date, amount)
VALUES (transactions_seq.NEXTVAL, 1, 1, DATE '2024-01-15', 25000);

INSERT INTO transactions (transaction_id, customer_id, product_id, sale_date, amount)
VALUES (transactions_seq.NEXTVAL, 2, 2, DATE '2024-01-20', 500000);

COMMIT;



 /* ======== STEP 4: WINDOW FUNCTION IMPLEMENTATIONS ======== */

-- 4.1 Ranking – Top N Customers by Revenue
SELECT 
  c.customer_id,
  c.name,
  c.region,
  SUM(t.amount) AS total_revenue,
  ROW_NUMBER() OVER(ORDER BY SUM(t.amount) DESC) AS row_num,
  RANK()       OVER(ORDER BY SUM(t.amount) DESC) AS rank_num,
  DENSE_RANK() OVER(ORDER BY SUM(t.amount) DESC) AS dense_rank_num,
  PERCENT_RANK() OVER(ORDER BY SUM(t.amount) DESC) AS percent_rank
FROM transactions t
JOIN customers c ON t.customer_id = c.customer_id
GROUP BY c.customer_id, c.name, c.region;

-- 4.2 Aggregate – Running Totals and Trends
SELECT 
  c.region,
  TO_CHAR(t.sale_date,'YYYY-MM') AS month,
  SUM(t.amount) AS monthly_sales,
  SUM(SUM(t.amount)) OVER(
        PARTITION BY c.region 
        ORDER BY TO_CHAR(t.sale_date,'YYYY-MM')
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total,
  AVG(SUM(t.amount)) OVER(
        PARTITION BY c.region 
        ORDER BY TO_CHAR(t.sale_date,'YYYY-MM')
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg_3months,
  MIN(SUM(t.amount)) OVER(PARTITION BY c.region) AS min_month,
  MAX(SUM(t.amount)) OVER(PARTITION BY c.region) AS max_month
FROM transactions t
JOIN customers c ON t.customer_id = c.customer_id
GROUP BY c.region, TO_CHAR(t.sale_date,'YYYY-MM');

-- 4.3 Navigation – Period-to-Period Growth
SELECT 
  c.region,
  TO_CHAR(t.sale_date,'YYYY-MM') AS month,
  SUM(t.amount) AS monthly_sales,
  LAG(SUM(t.amount)) OVER(
        PARTITION BY c.region 
        ORDER BY TO_CHAR(t.sale_date,'YYYY-MM')) AS prev_month_sales,
  (SUM(t.amount) - 
   LAG(SUM(t.amount)) OVER(PARTITION BY c.region ORDER BY TO_CHAR(t.sale_date,'YYYY-MM'))) /
   NULLIF(LAG(SUM(t.amount)) OVER(PARTITION BY c.region ORDER BY TO_CHAR(t.sale_date,'YYYY-MM')),0) * 100 AS growth_percent
FROM transactions t
JOIN customers c ON t.customer_id = c.customer_id
GROUP BY c.region, TO_CHAR(t.sale_date,'YYYY-MM');

-- 4.4 Distribution – Customer Segmentation
SELECT 
  c.customer_id,
  c.name,
  c.region,
  SUM(t.amount) AS total_spent,
  NTILE(4) OVER(ORDER BY SUM(t.amount) DESC) AS spending_quartile,
  CUME_DIST() OVER(ORDER BY SUM(t.amount) DESC) AS cumulative_distribution
FROM transactions t
JOIN customers c ON t.customer_id = c.customer_id
GROUP BY c.customer_id, c.name, c.region;



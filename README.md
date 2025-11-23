# Danny's Dinner SQL Quiz - Complete Guide

## 📊 Database Schema

### Table: `sales`
| Column | Type | Description |
|--------|------|-------------|
| customer_id | VARCHAR | Customer identifier (A, B, C) |
| order_date | DATE | When the order was placed |
| productid | INTEGER | Product ordered (FK to menu) |

### Table: `menu`
| Column | Type | Description |
|--------|------|-------------|
| productid | INTEGER | Unique product identifier (PK) |
| product_name | VARCHAR | Item name (sushi, curry, ramen) |
| price | INTEGER | Price in dollars |

### Table: `members`
| Column | Type | Description |
|--------|------|-------------|
| customer_id | VARCHAR | Customer identifier (PK) |
| join_date | DATE | Membership enrollment date |

### Relationships
- `sales.productid` → `menu.productid`
- `sales.customer_id` → `members.customer_id` (LEFT JOIN - not all customers are members)

---

## 📝 Questions & Solutions

### Question 1: Total Amount Each Customer Spent

**Task:** What is the total amount each customer spent at the restaurant?

**SQL Query:**
```sql
SELECT DISTINCT s.customer_id, SUM(m.price) AS totalamount
FROM sales s
JOIN menu m ON s.productid = m.productid
GROUP BY customer_id;
```

**Result:**
| customer_id | totalamount |
|-------------|-------------|
| A | 76 |
| B | 74 |
| C | 36 |

**Explanation:** Use JOIN between sales and menu, then SUM(price) grouped by customer to compute total spend per customer. Useful to compute lifetime value.

---

### Question 2: Days Each Customer Visited

**Task:** How many days has each customer visited the restaurant?

**SQL Query:**
```sql
SELECT customer_id, COUNT(DISTINCT(order_date)) AS no_f_days_visited
FROM sales
GROUP BY customer_id;
```

**Result:**
| customer_id | no_f_days_visited |
|-------------|-------------------|
| A | 4 |
| B | 6 |
| C | 2 |

**Explanation:** Counting distinct dates gives number of unique visit days (multiple orders same day count once). Useful for retention/visits analysis.

---

### Question 3: First Item Purchased by Each Customer

**Task:** What was the first item from the menu purchased by each customer?

**SQL Query:**
```sql
WITH cte AS (
  SELECT s.customer_id, m.product_name,
         ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY s.order_date) AS rownum
  FROM sales s
  JOIN menu m ON s.productid = m.productid
)
SELECT customer_id, product_name
FROM cte
WHERE rownum = 1;
```

**Result:**
| customer_id | product_name |
|-------------|--------------|
| A | sushi |
| B | curry |
| C | ramen |

**Explanation:** ROW_NUMBER over (partition by customer order by date) picks earliest purchase per customer.

---

### Question 4: Most Purchased Item on Menu

**Task:** What is the most purchased item on the menu and how many times was it purchased?

**SQL Query:**
```sql
SELECT m.product_name, COUNT(*) AS odr_count
FROM sales s
JOIN menu m ON s.productid = m.productid
GROUP BY m.product_name
ORDER BY odr_count DESC
LIMIT 1;
```

**Result:**
| product_name | odr_count |
|--------------|-----------|
| ramen | 8 |

**Explanation:** Ramen was ordered 8 times across all customers — useful for inventory & popular item promotions.

---

### Question 5: Most Popular Item per Customer

**Task:** Which item was the most popular for each customer?

**SQL Query:**
```sql
WITH cte AS (
  SELECT s.customer_id, m.product_name, COUNT(*) AS order_count,
         DENSE_RANK() OVER (PARTITION BY s.customer_id ORDER BY COUNT(*) DESC) AS rn
  FROM sales s
  JOIN menu m ON s.productid = m.productid
  GROUP BY s.customer_id, m.product_name
)
SELECT customer_id, product_name
FROM cte
WHERE rn = 1;
```

**Result:**
| customer_id | product_name |
|-------------|--------------|
| A | ramen |
| B | curry |
| C | ramen |

**Explanation:** Per-customer popularity helps personalized recommendations and loyalty targeting.

---

### Question 6: First Item After Becoming Member

**Task:** Which item was purchased first by the customer after they became a member?

**SQL Query:**
```sql
WITH orders AS (
  SELECT s.customer_id, m.product_name, s.order_date, mb.join_date,
         DENSE_RANK() OVER (PARTITION BY s.customer_id ORDER BY s.order_date) AS rn
  FROM menu m
  JOIN sales s ON s.productid = m.productid
  JOIN members mb ON s.customer_id = mb.customer_id
  WHERE s.order_date > mb.join_date
)
SELECT customer_id, product_name
FROM orders
WHERE rn = 1;
```

**Result:**
| customer_id | product_name |
|-------------|--------------|
| A | ramen |
| B | sushi |

**Explanation:** Identify first purchase after enrollment — useful to track welcome-offer usage.

---

### Question 7: Item Purchased Before Becoming Member

**Task:** Which item was purchased just before the customer became a member?

**SQL Query:**
```sql
WITH cte AS (
  SELECT s.customer_id, m.product_name, s.order_date, mb.join_date,
         DENSE_RANK() OVER (PARTITION BY s.customer_id ORDER BY s.order_date DESC) AS rn
  FROM menu m
  JOIN sales s ON s.productid = m.productid
  JOIN members mb ON s.customer_id = mb.customer_id
  WHERE s.order_date < mb.join_date
)
SELECT customer_id, product_name
FROM cte
WHERE rn = 1;
```

**Result:**
| customer_id | product_name |
|-------------|--------------|
| A | curry |
| B | sushi |

**Explanation:** Find order immediately before member enrollment — useful to analyze conversion behavior.

---

### Question 8: Pre-Membership Spending

**Task:** Total items and amount spent for each member before they became a member

**SQL Query:**
```sql
SELECT s.customer_id, COUNT(m.productid) AS items_ordered, SUM(price) AS total_amount
FROM menu m
JOIN sales s ON m.productid = s.productid
JOIN members mb ON s.customer_id = mb.customer_id
WHERE s.order_date < mb.join_date
GROUP BY customer_id;
```

**Result:**
| customer_id | items_ordered | total_amount |
|-------------|---------------|--------------|
| A | 2 | 25 |
| B | 3 | 40 |

**Explanation:** Shows how much each member spent before joining — can help analyze the lift after joining.

---

### Question 9: Loyalty Points Calculation

**Task:** If each $1 spent = 10 points and sushi has a 2x multiplier — how many points would each customer have?

**SQL Query:**
```sql
SELECT customer_id, 
       SUM(CASE 
         WHEN product_name = 'sushi' THEN price * 20
         ELSE price * 10
       END) AS points
FROM sales
JOIN menu USING(productid)
GROUP BY customer_id;
```

**Result:**
| customer_id | points |
|-------------|--------|
| A | 860 |
| B | 940 |
| C | 360 |

**Explanation:** Each $1 = 10 points; sushi doubles points. This query is useful to compute loyalty points.

---

### Question 10: First Week Bonus Points

**Task:** In the first week after a customer joins they earn 2x points on all items (including join date). How many points do customers A and B have at the end of January?

**SQL Query:**
```sql
WITH cte AS (
  SELECT s.customer_id, m.product_name, s.order_date, mb.join_date,
         CASE
           WHEN s.order_date BETWEEN mb.join_date AND DATE_ADD(mb.join_date, INTERVAL 7 DAY) 
                THEN m.price * 10 * 2
           WHEN m.product_name = 'sushi' THEN m.price * 10 * 2
           ELSE m.price * 10
         END AS points
  FROM menu m
  JOIN sales s ON s.productid = m.productid
  JOIN members mb ON s.customer_id = mb.customer_id
  WHERE s.order_date < '2021-02-01'
)
SELECT customer_id, SUM(points) AS total_points
FROM cte
GROUP BY customer_id;
```

**Result:**
| customer_id | total_points |
|-------------|--------------|
| A | 1370 |
| B | 940 |

**Explanation:** First-week double points dramatically increases points for new members — track for promotion effectiveness.

---

### Question 11: Member Status per Order

**Task:** Show product name & price for each order date and indicate whether the customer was a member on that order date.

**SQL Query:**
```sql
SELECT s.customer_id, m.product_name, m.price, s.order_date,
       CASE 
         WHEN s.order_date >= mb.join_date THEN 'Y'
         ELSE 'N'
       END AS member
FROM menu m
JOIN sales s ON s.productid = m.productid
LEFT JOIN members mb ON mb.customer_id = s.customer_id;
```

**Result:**
| customer_id | product_name | price | order_date | member |
|-------------|--------------|-------|------------|--------|
| A | sushi | 10 | 2021-01-01 | N |
| A | curry | 15 | 2021-01-01 | N |
| A | curry | 15 | 2021-01-07 | Y |
| A | ramen | 12 | 2021-01-10 | Y |
| A | ramen | 12 | 2021-01-11 | Y |
| A | ramen | 12 | 2021-01-11 | Y |
| B | curry | 15 | 2021-01-01 | N |
| B | curry | 15 | 2021-01-02 | N |
| B | sushi | 10 | 2021-01-04 | N |
| B | sushi | 10 | 2021-01-11 | Y |
| B | ramen | 12 | 2021-01-16 | Y |
| B | ramen | 12 | 2021-02-01 | Y |
| C | ramen | 12 | 2021-01-01 | N |
| C | ramen | 12 | 2021-01-01 | N |
| C | ramen | 12 | 2021-01-07 | N |

**Explanation:** A per-order membership flag helps analyze orders by member vs non-member.

---

## 🎯 Key SQL Concepts Covered

1. **JOINs** - INNER JOIN, LEFT JOIN for combining tables
2. **Aggregations** - SUM, COUNT, GROUP BY
3. **Window Functions** - ROW_NUMBER, DENSE_RANK, PARTITION BY
4. **CTEs (Common Table Expressions)** - WITH clause for complex queries
5. **CASE Statements** - Conditional logic in SELECT
6. **Date Functions** - DATE_ADD, date comparisons
7. **DISTINCT** - Removing duplicates
8. **Filtering** - WHERE clause with date conditions
9. **Sorting & Limiting** - ORDER BY, LIMIT

---

## 💡 Business Use Cases

- **Customer Lifetime Value (CLV)** - Track total spending per customer
- **Retention Analysis** - Monitor visit frequency
- **Product Popularity** - Identify best-sellers for inventory management
- **Personalization** - Recommend items based on customer preferences
- **Member Conversion** - Analyze behavior before/after membership
- **Loyalty Programs** - Calculate reward points with multipliers
- **Promotion Effectiveness** - Track first-week member bonuses

---

## 📚 Learning Resources

This quiz is based on Danny Ma's 8 Week SQL Challenge - Case Study #1: Danny's Diner

Practice these queries to master:
- Advanced SQL aggregations
- Window functions for ranking
- Multi-table joins
- Business analytics queries
- Customer behavior analysis

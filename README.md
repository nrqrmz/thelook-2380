# SQL Recap: Top 3 Categories by Revenue in Each Country

Open-ended recap exercise for the Data Analytics SQL module, using the `thelook_ecommerce` public dataset in BigQuery.

**Business question:** What are the top 3 categories with the highest revenue in each country?

**Tables used:** `order_items`, `products`, `users`

---

## 1. Understand the Problem

Before opening BigQuery, think about the problem.

- What do we need at the end? What would the final table look like?
- How many rows do we expect per country?
- Which tables do we need, and why?
- What steps would we follow to get there?

## 2. Granularity

- What does one row represent in `order_items`, `products`, and `users`?
- Which column connects each table to `order_items`?
- If we join the three tables, how many rows should the result have? How do we verify it?

## 3. Cleaning and Definitions

- Should a cancelled or returned order count as revenue? Check the values in `status`.
- Which column represents revenue?
- Are there nulls in `country` or `category`?

## 4. First CTE: Revenue by Country and Category

- Which columns do we group by?
- How do we check that the sums make sense before moving on?

## 5. Window Function: Ranking

- Can we filter `RANK() <= 3` directly in the `WHERE` clause? Why not?
- What goes in the `PARTITION BY` and what goes in the `ORDER BY`?
- What happens if two categories tie? `ROW_NUMBER`, `RANK`, or `DENSE_RANK`?

### ROW_NUMBER vs RANK vs DENSE_RANK

Given revenues of 100, 90, 90, and 80:

| Function | Result | Behavior on ties |
|---|---|---|
| `ROW_NUMBER` | 1, 2, 3, 4 | Always unique, tied rows get arbitrary order |
| `RANK` | 1, 2, 2, 4 | Ties share a position, next position is skipped |
| `DENSE_RANK` | 1, 2, 2, 3 | Ties share a position, no gaps |

We use `DENSE_RANK` here: if categories tie, none is left out of the top 3 by chance, and positions are not skipped.

In this dataset, revenue is a sum of decimals, so ties are very rare and all three functions would likely return the same result. To see the difference, rank by number of orders instead of revenue.

When to use each:

- `ROW_NUMBER`: when you need exactly one row per group, e.g. the latest order per customer, or deduplication.
- `RANK`: when the position should reflect how many are ahead, e.g. two tied for second, next is fourth.
- `DENSE_RANK`: when you want levels without gaps and ties allowed, e.g. the top 3 distinct values.

## 6. Wrap-up

- How do we know the result is correct? Validate by summing one country by hand and checking that each country has at most three rows.
- Which part would you turn into a view, and why?

---

## Solution

Replace `your-project.your_dataset` with your own project and dataset.

```sql
WITH sales AS (
  -- Step 1: join the three tables and keep only valid revenue
  SELECT
    u.country,
    p.category,
    oi.sale_price
  FROM `your-project.your_dataset.order_items` AS oi
  JOIN `your-project.your_dataset.products` AS p
    ON oi.product_id = p.id
  JOIN `your-project.your_dataset.users` AS u
    ON oi.user_id = u.id
  WHERE oi.status NOT IN ('Cancelled', 'Returned')
),

category_revenue AS (
  -- Step 2: revenue by country and category
  SELECT
    country,
    category,
    ROUND(SUM(sale_price), 2) AS revenue
  FROM sales
  GROUP BY country, category
),

ranked AS (
  -- Step 3: rank categories within each country
  SELECT
    country,
    category,
    revenue,
    DENSE_RANK() OVER (PARTITION BY country ORDER BY revenue DESC) AS revenue_rank
  FROM category_revenue
)

-- Step 4: keep the top 3 per country
SELECT
  country,
  category,
  revenue,
  revenue_rank
FROM ranked
WHERE revenue_rank <= 3
ORDER BY country, revenue_rank;
```

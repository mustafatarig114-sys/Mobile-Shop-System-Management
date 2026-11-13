# Mobile Shop – SQL Joins & Analysis Report

A practical project on the **Mobile shop** database using **PostgreSQL 18** (via pgAdmin). This README documents the queries that were run, the screenshots of their results, and a short report on the findings.

## Project Structure
---202510537
```
.
├── README.md
└── images/
    ├── 01-inner-join.png
    ├── 02-left-join.png
    ├── 03-right-join.png
    └── 04-moving-average.png
```

## Tables Used

| Table | Columns used |
|---|---|
| `customers` | `customer_id`, `customer_name` |
| `sales` | `sale_id`, `customer_id`, `sale_date`, `total_amount` |
| `employees` | `employee_name`, `branch_id` |
| `branches` | `branch_id`, `branch_name` |

---

## 1) INNER JOIN – Customers who have sales only

An INNER JOIN keeps only the rows that have a match on both sides, so only customers with at least one sale appear.

```sql
SELECT
    customers.customer_name,
    sales.sale_id,
    sales.total_amount
FROM customers
INNER JOIN sales
ON customers.customer_id = sales.customer_id;
```

**Result:** 11 rows (query completed in about 0.104 s).

![INNER JOIN between customers and sales in pgAdmin, returning 11 rows](images/01-inner-join.png)

*Screenshot 1 – INNER JOIN of `customers` and `sales`. The Data Output shows `customer_name`, `sale_id` and `total_amount`, with "Total rows: 11".*

| customer_name | sale_id | total_amount |
|---|---|---|
| Ahmed Ali | 1 | 925000.00 |
| Mary John | 2 | 1200000.00 |
| James Peter | 3 | 330000.00 |
| Ahmed Ali | 4 | 400000.00 |
| Sarah David | 5 | 1280000.00 |
| Michael Paul | 6 | 250000.00 |
| Mary John | 7 | 500000.00 |
| James Peter | 8 | 1200000.00 |
| Sarah David | 9 | 900000.00 |

> The first 9 of the 11 rows are visible in the screenshot.

---

## 2) LEFT JOIN – All customers, including those with no sales

A LEFT JOIN keeps every row from the left table (`customers`), even when there is no matching sale.

```sql
SELECT
    customers.customer_name,
    sales.sale_id
FROM customers
LEFT JOIN sales
ON customers.customer_id = sales.customer_id;
```

**Result:** 12 rows (compared to 11 with the INNER JOIN in Screenshot 1). This means one extra row comes from `customers` with no matching sale.

![LEFT JOIN between customers and sales in pgAdmin, returning 12 rows](images/02-left-join.png)

*Screenshot 2 – LEFT JOIN of `customers` and `sales`. "Total rows: 12" is one more than the INNER JOIN, which is how the unmatched customer shows up.*

---

## 3) RIGHT JOIN – All branches with their employees

A RIGHT JOIN keeps every row from the right table (`branches`), even when no employee is assigned to it.

```sql
SELECT
    employees.employee_name,
    branches.branch_name
FROM employees
RIGHT JOIN branches
ON employees.branch_id = branches.branch_id;
```

**Result:** 5 rows.

![RIGHT JOIN between employees and branches in pgAdmin, returning 5 rows with a null employee for Huye Branch](images/03-right-join.png)

*Screenshot 3 – RIGHT JOIN of `employees` and `branches`. Row 5 shows `[null]` for the employee because Huye Branch has no employees.*

| employee_name | branch_name |
|---|---|
| John Manager | Kigali Central |
| Peter Sales | Kigali Central |
| David Manager | Musanze Branch |
| Samuel Sales | Musanze Branch |
| [null] | Huye Branch |

---

## 4) FULL OUTER JOIN – Customers and sales (everything from both sides)

A FULL OUTER JOIN returns every customer and every sale, whether or not they have a match on the other side.

```sql
SELECT
    customers.customer_name,
    sales.sale_id,
    sales.total_amount
FROM customers
FULL OUTER JOIN sales
ON customers.customer_id = sales.customer_id;
```

This query is visible in the editor of Screenshot 3 (lines 7–13), written right after the RIGHT JOIN query. The Data Output in that screenshot belongs to the RIGHT JOIN, so the FULL OUTER JOIN result itself is not captured in the screenshots.

![pgAdmin query editor showing the RIGHT JOIN and FULL OUTER JOIN queries](images/03-right-join.png)

*Screenshot 3 (again) – the editor shows the RIGHT JOIN query (lines 1–6) followed by the FULL OUTER JOIN query (lines 7–13).*

---

## 5) Moving Average – 3-month average revenue

This query groups sales by month with `DATE_TRUNC`, then uses a window function to average each month with the two months before it.

```sql
WITH monthly_sales AS (
    SELECT
        DATE_TRUNC('month', sale_date) AS sales_month,
        SUM(total_amount) AS monthly_revenue
    FROM sales
    GROUP BY DATE_TRUNC('month', sale_date)
)
SELECT
    sales_month,
    monthly_revenue,
    AVG(monthly_revenue) OVER (
        ORDER BY sales_month
        ROWS BETWEEN 2 PRECEDING
        AND CURRENT ROW
    ) AS three_month_average
FROM monthly_sales;
```

**Result:** 6 rows.

![3-month moving average of monthly revenue in pgAdmin, returning 6 rows](images/04-moving-average.png)

*Screenshot 4 – the `monthly_sales` CTE and the window function in the editor (lines 145–158), with the 6 monthly rows in the Data Output.*

| sales_month | monthly_revenue | three_month_average |
|---|---|---|
| 2026-01-01 | 2,125,000.00 | 2,125,000.00 |
| 2026-02-01 | 730,000.00 | 1,427,500.00 |
| 2026-03-01 | 1,530,000.00 | 1,461,666.67 |
| 2026-04-01 | 1,700,000.00 | 1,320,000.00 |
| 2026-05-01 | 1,380,000.00 | 1,536,666.67 |
| 2026-06-01 | 1,305,000.00 | 1,461,666.67 |

---

# Report

## Summary of Findings

1. **Customers and sales (Screenshots 1 and 2):** The INNER JOIN returned 11 rows while the LEFT JOIN returned 12, so at least one customer is registered but has never made a purchase. This is an opportunity for a marketing campaign to encourage a first purchase.
2. **Repeat customers (Screenshot 1):** Customers such as Ahmed Ali, Mary John, James Peter and Sarah David have more than one purchase, which indicates good customer loyalty.
3. **Largest sales (Screenshot 1):** Among the visible rows, the highest sale was **1,280,000** (Sarah David), followed by **1,200,000** (Mary John and James Peter).
4. **Branches and employees (Screenshot 3):** **Huye Branch** has no employee assigned (NULL), while Kigali Central and Musanze Branch each have a manager and a sales employee. Staffing or branch data should be reviewed.
5. **Monthly revenue, January – June 2026 (Screenshot 4):**
   - Total revenue: **8,770,000**
   - Average monthly revenue: about **1,461,667**
   - Best month: **January (2,125,000)**
   - Weakest month: **February (730,000)**, a sharp drop of about 66% after January.
   - From March to June, revenue is relatively stable between 1.3 and 1.7 million.
6. **Moving average (Screenshot 4):** The trend is steady at around 1.3 – 1.5 million per month after the January peak. The first two months are averaged over fewer than 3 months of data (January alone, then January and February).

## Recommendations

- Follow up with customers who have no sales and offer them promotions.
- Address Huye Branch (assign employees or update the data).
- Investigate the cause of the February drop and try to repeat January's performance (seasonal offers or campaigns).
- Use the moving average as a regular indicator to monitor the sales trend.

## Tools

- PostgreSQL 18
- pgAdmin 4

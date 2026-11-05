# Walmart Sales Data Analysis: End-to-End Python + SQL Project

## Overview
This project is an end-to-end data analysis of Walmart sales transactions. Python (Pandas, SQLAlchemy, psycopg2) is used to load and clean the raw data, which is then loaded into PostgreSQL, where SQL is used to answer real business questions. This README documents the objectives, workflow, business problems, solutions, findings, and conclusions.

## Objectives

- Clean and prepare raw sales data using Python and Pandas.
- Load the cleaned data into a relational database (PostgreSQL / MySQL).
- Analyze payment methods, category ratings, branch traffic, and profit.
- Identify branch-level trends such as busiest days, shift activity, and year-over-year revenue decline.
- Practice `GROUP BY`, aggregate functions, window functions, CTEs, and date functions.

## Project Workflow

```
Kaggle Dataset  ->  Python (Process, Clean, Load)  ->  PostgreSQL / MySQL  ->  Solve Business Problems
```

**Tools used:** Pandas, SQLAlchemy, PyMySQL, Psycopg2, IPython / Jupyter, PostgreSQL

## Dataset

The data for this project is sourced from Kaggle:

- **Raw file:** `Walmart.csv` (10,051 rows, 11 columns)
- **Cleaned file:** `walmart_clean_data.csv` (9,969 rows, 12 columns)

## Data Cleaning Steps (Python)

All steps are in `project.ipynb`:

1. Loaded `Walmart.csv` with Pandas and inspected it using `head()`, `describe()`, and `info()`.
2. Removed **51 duplicate rows** (10,051 -> 10,000).
3. Dropped **31 rows with missing values** in `unit_price` and `quantity` (10,000 -> 9,969).
4. Converted `unit_price` from text (e.g. `$74.69`) to a float by removing the `$` sign.
5. Created a new column `total = unit_price * quantity`.
6. Converted all column names to lowercase.
7. Saved the result to `walmart_clean_data.csv`.
8. Loaded the cleaned data into PostgreSQL using SQLAlchemy (`df.to_sql`).

## Schema

After loading, the `walmart` table has the following columns:

| Column | Description |
|---|---|
| invoice_id | Unique transaction ID |
| branch | Branch code (e.g. WALM003) |
| city | City of the branch |
| category | Product category |
| unit_price | Price per unit |
| quantity | Number of units sold |
| date | Transaction date (`DD/MM/YY`) |
| time | Transaction time |
| payment_method | Cash, Credit card, or Ewallet |
| rating | Customer rating |
| profit_margin | Profit margin of the sale |
| total | `unit_price * quantity` |

```sql
-- Verify the data load
SELECT * FROM walmart;
SELECT COUNT(*) FROM walmart;
```

> Adjust column names and types if your imported data differs.

## Business Problems and Solutions

### 1. Analyze Payment Methods and Sales

```sql
SELECT
    payment_method,
    COUNT(*) AS no_payments,
    SUM(quantity) AS no_qty_sold
FROM walmart
GROUP BY payment_method;
```

**Objective:** Find the different payment methods, with the number of transactions and items sold for each. This shows customer payment preferences.

### 2. Identify the Highest-Rated Category in Each Branch

```sql
SELECT *
FROM
(
    SELECT
        branch,
        category,
        AVG(rating) AS avg_rating,
        RANK() OVER(PARTITION BY branch ORDER BY AVG(rating) DESC) AS ranking
    FROM walmart
    GROUP BY 1, 2
) AS t
WHERE ranking = 1;
```

**Objective:** Find the category with the highest average rating in each branch, so popular categories can be promoted locally.

### 3. Determine the Busiest Day for Each Branch

```sql
SELECT *
FROM
(
    SELECT
        branch,
        TRIM(TO_CHAR(TO_DATE(date, 'DD/MM/YY'), 'Day')) AS day_name,
        COUNT(*) AS no_transactions,
        RANK() OVER(PARTITION BY branch ORDER BY COUNT(*) DESC) AS ranking
    FROM walmart
    GROUP BY 1, 2
) AS t
WHERE ranking = 1;
```

**Objective:** Find the weekday with the most transactions for each branch, to help with staffing and inventory planning.

### 4. Calculate Total Quantity Sold by Payment Method

```sql
SELECT
    payment_method,
    SUM(quantity) AS total_quantity
FROM walmart
GROUP BY payment_method;
```

**Objective:** Track sales volume by payment type to understand customer purchasing habits.

### 5. Analyze Category Ratings by City

```sql
SELECT
    city,
    category,
    MIN(rating) AS min_rating,
    MAX(rating) AS max_rating,
    AVG(rating) AS avg_rating
FROM walmart
GROUP BY 1, 2;
```

**Objective:** Get the minimum, maximum, and average rating of each category in each city, to guide city-level promotions.

### 6. Calculate Total Profit by Category

```sql
SELECT
    category,
    SUM(total) AS total_revenue,
    SUM(total * profit_margin) AS total_profit
FROM walmart
GROUP BY 1
ORDER BY 3 DESC;
```

**Objective:** Calculate revenue and profit (`total * profit_margin`) per category, ranked from highest to lowest profit.

### 7. Determine the Most Common Payment Method per Branch

```sql
WITH cte AS
(
    SELECT
        branch,
        payment_method,
        COUNT(*) AS total_trans,
        RANK() OVER(PARTITION BY branch ORDER BY COUNT(*) DESC) AS ranking
    FROM walmart
    GROUP BY branch, payment_method
)
SELECT
    branch,
    payment_method AS preferred_payment_method
FROM cte
WHERE ranking = 1;
```

**Objective:** Find the preferred payment method of each branch, so branches can streamline their payment processing.

### 8. Analyze Sales Shifts Throughout the Day

```sql
SELECT
    branch,
    CASE
        WHEN EXTRACT(HOUR FROM time::time) < 12 THEN 'Morning'
        WHEN EXTRACT(HOUR FROM time::time) BETWEEN 12 AND 17 THEN 'Afternoon'
        ELSE 'Evening'
    END AS shift,
    COUNT(*) AS no_transactions
FROM walmart
GROUP BY 1, 2
ORDER BY 1, 3 DESC;
```

**Objective:** Count transactions in each shift (Morning, Afternoon, Evening) per branch, to plan staff shifts and stock replenishment.

### 9. Identify Branches with the Highest Revenue Decline Year-over-Year

```sql
WITH revenue_2022 AS
(
    SELECT
        branch,
        SUM(total) AS revenue
    FROM walmart
    WHERE EXTRACT(YEAR FROM TO_DATE(date, 'DD/MM/YY')) = 2022
    GROUP BY 1
),
revenue_2023 AS
(
    SELECT
        branch,
        SUM(total) AS revenue
    FROM walmart
    WHERE EXTRACT(YEAR FROM TO_DATE(date, 'DD/MM/YY')) = 2023
    GROUP BY 1
)
SELECT
    ls.branch,
    ls.revenue AS last_year_revenue,
    cs.revenue AS current_year_revenue,
    ROUND(
        (ls.revenue - cs.revenue)::numeric / ls.revenue::numeric * 100,
        2
    ) AS rev_dec_ratio
FROM revenue_2022 AS ls
JOIN revenue_2023 AS cs
    ON ls.branch = cs.branch
WHERE ls.revenue > cs.revenue
ORDER BY 4 DESC
LIMIT 5;
```

**Objective:** Find the 5 branches whose revenue fell the most from 2022 to 2023, to spot possible local issues.

> For MySQL, replace `EXTRACT(YEAR FROM TO_DATE(...))` with `YEAR(STR_TO_DATE(date, '%d/%m/%y'))`.

## SQL Concepts Used

- Aggregation: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `GROUP BY`, `ORDER BY`, `LIMIT`
- Window functions: `RANK() OVER (PARTITION BY ...)`
- Subqueries and CTEs (`WITH`)
- Conditional logic: `CASE WHEN`
- Date and time functions: `TO_DATE`, `TO_CHAR`, `EXTRACT`
- Type casting: `::numeric`, `::time`

## Findings and Conclusion

- **Payments:** Comparing transactions and quantity by payment method shows which payment options customers rely on most.
- **Ratings:** Highest-rated categories per branch and rating ranges per city show where customer satisfaction is strongest or weakest.
- **Traffic Patterns:** Busiest days and shift counts show when branches need the most staff and stock.
- **Profitability:** Profit by category highlights which product lines contribute most to earnings.
- **Revenue Trends:** The year-over-year comparison flags branches with declining revenue that need attention.

This analysis shows how Python and SQL work together, from raw data to business insight, and can help inform staffing, inventory, marketing, and pricing decisions.

## How to Run

1. Install the dependencies: `pip install pandas sqlalchemy pymysql psycopg2-binary`
2. Download `Walmart.csv` from Kaggle and place it in the project folder.
3. Create a PostgreSQL database named `Walmart`.
4. Run `project.ipynb` to clean the data and load it into the `walmart` table. Enter your database password when prompted. Do not hardcode it in the notebook.
5. Run the queries in `Process.sql`.

## Project Structure

```
├── Walmart.csv               # Raw dataset
├── walmart_clean_data.csv    # Cleaned dataset
├── project.ipynb             # Data cleaning and loading (Python)
├── Process.sql               # SQL queries for business problems
├── requirements.txt
└── README.md
```

## Author - [Your Name]

This project is part of my portfolio, showcasing the Python and SQL skills essential for data analyst roles. If you have any questions, feedback, or would like to collaborate, feel free to get in touch!

- **YouTube**: [Add your channel link]
- **Instagram**: [Add your profile link]
- **LinkedIn**: [Add your profile link]
- **GitHub**: [Add your profile link]

Thank you for your support, and I look forward to connecting with you!

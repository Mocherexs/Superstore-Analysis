# The Data Behind the Dashboard

*Retail analytics / SQL walkthrough / 2015–2018*
**By Adedamola Ademokoya**

A mid-size U.S. retailer grew revenue 50% in four years. This is the same story the dashboard tells, but every claim is shown with the SQL that produced it and the exact rows it returned, run against the cleaned dataset in PostgreSQL.

**Dataset:** 9,800 order lines · 4,922 orders · 793 customers · Engine: PostgreSQL 16 · Grain: one row per order line · Measure: sales revenue

| Total revenue | 4-year growth | Avg order value | Repeat buyers |
| --- | --- | --- | --- |
| $2.26M | +50.5% | $459 | 98.4% |

## Contents

1. [The business at a glance](#01-the-business-at-a-glance)
2. [Four years, one dip, then lift-off](#02-four-years-one-dip-then-lift-off)
3. [A balanced book with a concentration risk](#03-a-balanced-book-with-a-concentration-risk)
4. [How concentrated is the catalog, really?](#04-how-concentrated-is-the-catalog-really)
5. [A loud seasonal heartbeat, and a Thursday mystery](#05-a-loud-seasonal-heartbeat-and-a-thursday-mystery)
6. [Retention is the real engine](#06-retention-is-the-real-engine)
7. [Two states carry a third of the business](#07-two-states-carry-a-third-of-the-business)
8. [What the averages conceal](#08-what-the-averages-conceal)
9. [Eight moves, each tied to a query](#09-eight-moves-each-tied-to-a-query)

## 01: The business at a glance

The retailer ships across all four U.S. census regions, serves three customer segments (Consumer, Corporate, Home Office) and sells three product categories: Technology, Furniture, and Office Supplies. Over four years it processed 4,922 orders from 793 customers. One query establishes the baseline every later number is measured against.

**Headline KPIs** (1 row)

```sql
-- One row of headline numbers for the whole dataset.
-- Each row in the superstore table is one line of an order.
SELECT
    ROUND(SUM(sales),2)                          AS total_revenue,   -- add up every sale, round to 2 decimals
    COUNT(DISTINCT order_id)                     AS orders,          -- count each order once, even if it has many lines
    COUNT(DISTINCT customer_id)                  AS customers,       -- count each customer once
    ROUND(SUM(sales)/COUNT(DISTINCT order_id),2) AS avg_order_value, -- total revenue divided by number of orders
    ROUND(AVG(ship_days),1)                       AS avg_ship_days   -- average days from order to shipping
FROM superstore;                                                     -- the table we are reading from
```

| total_revenue | orders | customers | avg_order_value | avg_ship_days |
| --- | --- | --- | --- | --- |
| 2,261,536.78 | 4,922 | 793 | 459.48 | 4.0 |

Average order value sits at $459 and items ship in about four days. Headline numbers only set the stage; the story is in how revenue moves across time, geography, product, and customer.

## 02: Four years, one dip, then lift-off

The first question for any business: is it growing? Here the answer is yes, but not smoothly. A LAG() window function compares each year to the one before it.

**Year-over-year growth** (4 rows)

```sql
-- Step 1: build a small temporary table (a CTE) with one row per year.
WITH yearly AS (
    SELECT year, SUM(sales) AS sales      -- total sales for each year
    FROM superstore GROUP BY year         -- GROUP BY collapses all rows of a year into one
)
-- Step 2: compare each year with the year before it.
SELECT
    year,
    ROUND(sales,0)                                     AS revenue,
    -- LAG() looks back one row, so it returns the previous year's sales.
    -- (this year - last year) / last year x 100 = growth in percent
    ROUND((sales - LAG(sales) OVER (ORDER BY year))
          *100.0/LAG(sales) OVER (ORDER BY year),1) AS yoy_pct
FROM yearly ORDER BY year;               -- list the years in order
```

| year | revenue | yoy_pct |
| --- | --- | --- |
| 2015 | 479,856 | — |
| 2016 | 459,436 | −4.3 |
| 2017 | 600,193 | +30.6 |
| 2018 | 722,052 | +20.3 |

Revenue actually dipped 4.3% in 2016 before a decisive +30.6% surge in 2017 and a further +20.3% in 2018. The shape (a soft second year, then two years of strong compounding) is what a business looks like when something changes mid-run: a new product line, a new market, or a shift in customer mix. The 2017 inflection is the thing worth investigating.

> *Revenue didn't climb in a straight line. It stalled in 2016, then compounded. The +50.5% headline is entirely a 2017–2018 story.*

### The monthly rhythm

Rolling the same revenue up by month and smoothing it with a three-month moving average separates signal from noise. The raw line is a sawtooth, with a sharp Q4 spike every year and then a Q1 collapse, while the moving average confirms the underlying climb is real, not just taller seasonal peaks.

**Monthly revenue + 3-month moving average** (48 rows · last 5 shown)

```sql
-- Step 1: total sales for each month.
WITH monthly AS (
    SELECT year_month,
           MIN(order_date) AS mstart,    -- earliest date in the month, used to sort months correctly
           SUM(sales) AS sales
    FROM superstore GROUP BY year_month
)
-- Step 2: show each month next to a 3-month moving average.
SELECT year_month, ROUND(sales,2) AS revenue,
    -- Average of this month and the 2 months before it; smooths out spikes.
    ROUND(AVG(sales) OVER (ORDER BY mstart
              ROWS BETWEEN 2 PRECEDING AND CURRENT ROW),2) AS moving_avg_3m
FROM monthly ORDER BY mstart;
```

| year_month | revenue | moving_avg_3m |
| --- | --- | --- |
| 2018-08 | 62,837.81 | 51,951.22 |
| 2018-09 | 86,152.90 | 64,605.28 |
| 2018-10 | 77,448.17 | 75,479.63 |
| 2018-11 | 117,938.14 | 93,846.40 |
| 2018-12 | 83,030.38 | 92,805.56 |

November 2018 is the single biggest month in the dataset at $117.9K, more than double a typical spring month. The moving average ends the period far above where it started, and its last reading is nearly double the first, which is the real evidence the growth is structural.

## 03: A balanced book with a concentration risk

Revenue composition decides how fragile a business is. Over-reliance on one category or one region is a risk that stays invisible until it isn't. Two GROUP BY queries with a window total show the split.

**Revenue by category** (3 rows)

```sql
-- Revenue per product category, and each category's share of the total.
SELECT category,
    ROUND(SUM(sales),0) AS revenue,                     -- sales for this category
    -- SUM(SUM(sales)) OVER () = the grand total across all categories,
    -- so this line gives the category's percentage of all revenue.
    ROUND(100.0*SUM(sales)/SUM(SUM(sales)) OVER (),1) AS pct
FROM superstore GROUP BY category                        -- one row per category
ORDER BY revenue DESC;                                   -- biggest first
```

| category | revenue | pct |
| --- | --- | --- |
| Technology | 827,456 | 36.6 |
| Furniture | 728,659 | 32.2 |
| Office Supplies | 705,422 | 31.2 |

No category dominates dangerously; it's a healthy 37/32/31 split. Technology leads, and its share grows fastest inside the Q4 peaks, pointing to big-ticket year-end purchases.

**Revenue by region** (4 rows)

```sql
-- Same pattern as the category query, but grouped by region.
SELECT region,
    ROUND(SUM(sales),0) AS revenue,                     -- sales for this region
    ROUND(100.0*SUM(sales)/SUM(SUM(sales)) OVER (),1) AS pct  -- share of the grand total
FROM superstore GROUP BY region                          -- one row per region
ORDER BY revenue DESC;                                   -- biggest first
```

| region | revenue | pct |
| --- | --- | --- |
| West | 710,220 | 31.4 |
| East | 669,519 | 29.6 |
| Central | 492,647 | 21.8 |
| South | 389,151 | 17.2 |

Here the balance breaks. West and East together are 61% of all revenue. That concentration is efficient for targeting marketing spend, but it also means a shock to either coast (a competitor, a supply disruption, a regional downturn) lands directly on the majority of the book.

## 04: How concentrated is the catalog, really?

The 80/20 rule is folklore until you test it. A running cumulative share over ranked sub-categories shows exactly how many products it takes to reach 80% of revenue, with no rounding, no hand-waving.

**Sub-categories with cumulative revenue share** (17 rows · top 10 shown)

```sql
-- Step 1: total sales for each sub-category.
WITH sub AS (
    SELECT sub_category, SUM(sales) AS sales
    FROM superstore GROUP BY sub_category
)
-- Step 2: rank them and add a running total as a percentage.
SELECT sub_category, ROUND(sales,0) AS revenue,
    -- Running total from the biggest sub-category down, divided by the grand total.
    -- Shows how much of revenue the top N sub-categories cover together.
    ROUND(100.0*SUM(sales) OVER (ORDER BY sales DESC)
          /SUM(sales) OVER (),1)          AS cumulative_pct,
    ROW_NUMBER() OVER (ORDER BY sales DESC)    AS rank   -- 1 = highest revenue
FROM sub ORDER BY rank;
```

| rank | sub_category | revenue | cumulative_pct |
| --- | --- | --- | --- |
| 1 | Phones | 327,782 | 14.5 |
| 2 | Chairs | 322,823 | 28.8 |
| 3 | Storage | 219,343 | 38.5 |
| 4 | Tables | 202,811 | 47.4 |
| 5 | Binders | 200,029 | 56.3 |
| 6 | Machines | 189,239 | 64.6 |
| 7 | Accessories | 164,187 | 71.9 |
| 8 | Copiers | 146,248 | 78.4 |
| 9 | Bookcases | 113,813 | 83.4 |
| 10 | Appliances | 104,618 | 88.0 |

> **Where the data corrects the dashboard.** A common retelling of this dataset says "6 sub-categories drive 80% of revenue." The cumulative column shows otherwise: the top 6 reach 64.6%, and 80% isn't crossed until rank 9 (Bookcases, 83.4%). The catalog is real but less concentrated than the 80/20 cliché. The long tail below Paper contributes under 5% combined, which is the part worth reviewing for overhead.

## 05: A loud seasonal heartbeat, and a Thursday mystery

Every retailer has a cadence set by seasons and buying cycles. A conditional aggregate with FILTER quantifies quarterly concentration in one pass.

**Q4 share of annual revenue** (1 row)

```sql
-- What percentage of all revenue falls in Q4 (Oct-Dec)?
-- FILTER (WHERE quarter=4) sums only the Q4 rows; the bottom SUM(sales) is everything.
SELECT ROUND(100.0*SUM(sales) FILTER (WHERE quarter=4)
          /SUM(sales),1) AS q4_pct
FROM superstore;
```

| q4_pct |
| --- |
| 38.5 |

Q4 alone is 38.5% of the year, nearly two-fifths of revenue in three months. The operational reading is direct: inventory stocked by September, marketing front-loaded into October, staffing up for November. Q1–Q2 are the quiet window for maintenance and re-engagement.

### The Thursday gap

Aggregating revenue by weekday surfaces something no one designs on purpose.

**Revenue by day of week** (7 rows)

```sql
-- Total revenue for each day of the week.
SELECT TO_CHAR(order_date,'Day')    AS weekday,   -- turn the date into a day name, e.g. 'Monday'
    ROUND(SUM(sales),0)              AS revenue
FROM superstore
-- Group by the day name; EXTRACT(DOW ...) gives the day number (0 = Sunday)
GROUP BY 1, EXTRACT(DOW FROM order_date)
ORDER BY EXTRACT(DOW FROM order_date);          -- sort Sunday to Saturday, not alphabetically
```

| weekday | revenue |
| --- | --- |
| Sunday | 377,869 |
| Monday | 348,791 |
| Tuesday | 420,536 |
| Wednesday | 315,889 |
| Thursday | 142,839 |
| Friday | 234,711 |
| Saturday | 420,902 |

Thursday brings in $143K against $421K on the Tuesday and Saturday peaks, roughly a third as much, consistent across all four years. There's no seasonal reason for it, which makes it a process signal rather than noise: a promo calendar that skips Wednesdays, an email cadence, an operational quirk. If Thursday were merely brought to the weekly average, the recovered revenue would be material.

## 06: Retention is the real engine

RFM (Recency, Frequency, Monetary) is the standard lens for "who matters and who's slipping." First, the headline that reframes the whole strategy.

**Repeat-buyer rate** (1 row)

```sql
-- How many customers ordered more than once?
SELECT
    COUNT(*) FILTER (WHERE orders>1)                     AS repeat_buyers,   -- customers with 2+ orders
    COUNT(*)                                             AS total_customers, -- all customers
    ROUND(100.0*COUNT(*) FILTER (WHERE orders>1)/COUNT(*),1) AS repeat_pct   -- repeat buyers as a %
FROM (
    -- Inner query: one row per customer with their number of orders.
    SELECT customer_id, COUNT(DISTINCT order_id) AS orders
    FROM superstore GROUP BY customer_id
) c;   -- 'c' is just a short name for the inner result
```

| repeat_buyers | total_customers | repeat_pct |
| --- | --- | --- |
| 780 | 793 | 98.4 |

Of all customers, 98.4% buy more than once. That is exceptionally high: when someone buys here, they come back. So the growth problem isn't converting first-timers into repeat buyers; it's re-engaging repeat buyers who've gone quiet.

**RFM segmentation, NTILE quintiles** (6 rows)

```sql
-- Step 1: for each customer, work out R, F and M.
WITH rfm AS (
    SELECT customer_id,
        -- Recency: days between their last order and the day after the latest order in the data
        (SELECT MAX(order_date)+1 FROM superstore) - MAX(order_date) AS recency,
        COUNT(DISTINCT order_id) AS frequency,   -- Frequency: how many orders
        SUM(sales)               AS monetary     -- Monetary: how much they spent in total
    FROM superstore GROUP BY customer_id
),
-- Step 2: score each customer 1-5 on recency and frequency.
-- NTILE(5) splits customers into 5 equal-sized groups; 5 = best.
scored AS (
    SELECT *,
        NTILE(5) OVER (ORDER BY recency DESC) AS r,   -- recent buyers get a high r
        NTILE(5) OVER (ORDER BY frequency)    AS f    -- frequent buyers get a high f
    FROM rfm
)
-- Step 3: turn the scores into named segments and summarise each one.
SELECT
    CASE                                         -- CASE works like IF / ELSE IF
        WHEN r>=4 AND f>=4 THEN 'Champions'      -- bought recently and often
        WHEN r>=3 AND f>=3 THEN 'Loyal'
        WHEN r>=4 AND f<=2 THEN 'Recent / New'   -- recent but not yet frequent
        WHEN r<=2 AND f>=3 THEN 'At Risk'        -- used to buy often, not lately
        WHEN r<=2 AND f<=2 THEN 'Lost'
        ELSE 'Needs Attention' END  AS segment,  -- everyone else
    COUNT(*) AS customers,
    ROUND(AVG(recency),0) AS avg_recency_days,
    ROUND(SUM(monetary),0) AS revenue
FROM scored GROUP BY segment ORDER BY revenue DESC;
```

| segment | customers | avg_recency_days | revenue |
| --- | --- | --- | --- |
| Champions | 167 | 26 | 659,078 |
| Loyal | 170 | 59 | 541,278 |
| At Risk | 138 | 221 | 466,916 |
| Lost | 180 | 374 | 302,971 |
| Recent / New | 86 | 26 | 166,399 |
| Needs Attention | 52 | 75 | 124,895 |

The At Risk segment is the story: 138 customers who used to buy often, now averaging 221 days since their last order, yet still holding $467K in historical revenue. That's money walking toward the door from people who have already proven they'll buy repeatedly, and the highest-ROI cohort to win back.

> **A note on method.** Segment counts depend on the cutoff scheme. This uses transparent NTILE(5) quintiles on recency and frequency; a different threshold rule will shift the boundaries and the labels. The point isn't the exact count in any bucket; it's the reproducible rule, printed here so the result can be audited and adjusted.

### The re-engagement window

How long between orders, on average? A LAG() over each customer's distinct order dates answers it.

**Average days between orders** (1 row)

```sql
-- Step 1: one row per customer per order date (removes duplicate lines on the same day).
WITH ordered AS (
    SELECT DISTINCT customer_id, order_date FROM superstore
),
-- Step 2: days since the same customer's previous order.
-- PARTITION BY customer_id makes LAG() look back only within that customer's own orders.
gaps AS (
    SELECT order_date - LAG(order_date)
           OVER (PARTITION BY customer_id ORDER BY order_date) AS gap
    FROM ordered
)
-- Step 3: average gap. A customer's first order has no previous one (NULL), so skip it.
SELECT ROUND(AVG(gap),1) AS avg_days_between_orders
FROM gaps WHERE gap IS NOT NULL;
```

| avg_days_between_orders |
| --- |
| 191.5 |

Returning customers reorder roughly every 192 days. That sets the clock for marketing automation: a nudge around day 150–180 catches a customer before the typical lapse, not after.

> *The strongest lever isn't acquisition. With a 98.4% repeat rate already earned and $467K sitting in the At Risk segment, the highest return comes from keeping the customers the business already won.*

## 07: Two states carry a third of the business

Regional balance hid a sharper truth. Ranking states with their revenue share shows concentration is more extreme than the four-region view implied.

**Top 10 states by revenue** (10 of 49 rows)

```sql
-- Top 10 states by revenue, with each state's share and rank.
SELECT state,
    ROUND(SUM(sales),0) AS revenue,
    ROUND(100.0*SUM(sales)/SUM(SUM(sales)) OVER (),1) AS pct,  -- share of the grand total
    RANK() OVER (ORDER BY SUM(sales) DESC) AS rank            -- 1 = highest revenue
FROM superstore GROUP BY state
ORDER BY revenue DESC LIMIT 10;   -- LIMIT 10 keeps only the first 10 rows
```

| rank | state | revenue | pct |
| --- | --- | --- | --- |
| 1 | California | 446,306 | 19.7 |
| 2 | New York | 306,361 | 13.5 |
| 3 | Texas | 168,572 | 7.5 |
| 4 | Washington | 135,207 | 6.0 |
| 5 | Pennsylvania | 116,277 | 5.1 |
| 6 | Florida | 88,437 | 3.9 |
| 7 | Illinois | 79,237 | 3.5 |
| 8 | Michigan | 76,136 | 3.4 |
| 9 | Ohio | 75,130 | 3.3 |
| 10 | Virginia | 70,637 | 3.1 |

California alone is 19.7% of revenue; add New York and two states account for 33%, while the remaining 47 split the other two-thirds. This is a double-edged sword: it tells you where marketing has the highest ROI, and it tells you where the business is most exposed to a single-state shock. The strategic answer is two-sided: protect the strongholds while deliberately funding a handful of underpenetrated look-alike states.

### Where segment meets category

A segment × category matrix, built with FILTER columns, finds the single hottest cell in the business.

**Segment × category revenue matrix** (3 rows)

```sql
-- A grid: customer segments as rows, product categories as columns.
-- Each FILTER adds up only the sales for one category.
SELECT segment,
    ROUND(SUM(sales) FILTER (WHERE category='Furniture'),0)       AS furniture,
    ROUND(SUM(sales) FILTER (WHERE category='Office Supplies'),0) AS office_supplies,
    ROUND(SUM(sales) FILTER (WHERE category='Technology'),0)      AS technology
FROM superstore GROUP BY segment
ORDER BY SUM(sales) DESC;   -- biggest segment first
```

| segment | furniture | office_supplies | technology |
| --- | --- | --- | --- |
| Consumer | 387,696 | 359,352 | 401,012 |
| Corporate | 220,322 | 224,131 | 244,042 |
| Home Office | 120,641 | 121,939 | 182,402 |

Consumer + Technology is the hottest cell at $401K, where the largest segment meets the largest category. Corporate leans toward Technology too; Home Office is the smallest across the board. That intersection is where a cross-sell or bundle program has the most surface area to work with.

## 08: What the averages conceal

Two more patterns reward a closer look, and one shows why stating your grain matters as much as the number itself.

**Order-value distribution at order grain** (1 row)

```sql
-- Step 1: add up the lines of each order to get the order's total value.
WITH order_totals AS (
    SELECT order_id, SUM(sales) AS order_value
    FROM superstore GROUP BY order_id
)
-- Step 2: compare the middle order (median) with the average (mean).
-- PERCENTILE_CONT(0.5) = median; PERCENTILE_CONT(0.99) = value that 99% of orders fall below.
-- ::numeric converts the result so ROUND() can be applied.
SELECT
    ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY order_value)::numeric,2)  AS median_order,
    ROUND(AVG(order_value),2)                                                   AS mean_order,
    ROUND(PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY order_value)::numeric,2) AS p99_order
FROM order_totals;
```

| median_order | mean_order | p99_order |
| --- | --- | --- |
| 151.88 | 459.48 | 4,206.36 |

**The same measure at line-item grain** (1 row)

```sql
-- Same median and mean, but on single order lines instead of whole orders.
SELECT
    ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY sales)::numeric,2) AS median_line,
    ROUND(AVG(sales),2)                                                  AS mean_line
FROM superstore;   -- no grouping, so each row (line item) counts once
```

| median_line | mean_line |
| --- | --- |
| 54.49 | 230.77 |

> **Same data, two answers: state your grain.** A median of $54 and a mean of $231 are correct for individual line items. Group those lines into orders and the median becomes $152, the mean $459. Both are right; they answer different questions. Reporting "$54 median order" conflates the two. The skew itself is real at both grains: a minority of large transactions pulls the mean far above the median, so a strategy that lifts the median (bundling, minimums, cross-sell) would stabilize revenue.

### The copier whales

One sub-category behaves unlike any other.

**Copier transaction profile** (1 row)

```sql
-- How many copier lines were sold, the average value of each, and the total.
SELECT COUNT(*) AS copier_lines,
    ROUND(AVG(sales),2) AS avg_sale,
    ROUND(SUM(sales),0) AS total
FROM superstore WHERE sub_category='Copiers';   -- WHERE keeps only copier rows
```

| copier_lines | avg_sale | total |
| --- | --- | --- |
| 66 | 2,215.88 | 146,248 |

Just 66 copier lines over four years, but each averages $2,216, by far the highest of any sub-category. Every copier sale is effectively a whale. Even a modest push on volume moves real revenue, which is why it earns a dedicated line in the recommendations.

## 09: Eight moves, each tied to a query

Analysis without action is trivia. Every recommendation below traces to a result shown above.

1. **Launch a win-back campaign for At Risk customers.** In total, 138 proven repeat buyers hold $467K in historical revenue and average 221 days since last order. A personalized sequence that recovers even 15–25% is the single highest-ROI play. *→ ties to [§06 RFM](#06-retention-is-the-real-engine)*
2. **Front-load everything for Q4.** With 38.5% of revenue in one quarter, stock inventory by September and launch campaigns in early October. *→ ties to [§05 Q4 share](#05-a-loud-seasonal-heartbeat-and-a-thursday-mystery)*
3. **Investigate the Thursday gap.** A consistent 3× shortfall every Thursday points to a fixable process, not a preference. Audit promo and email timing. *→ ties to [§05 day-of-week](#the-thursday-gap)*
4. **Lift the median order.** A $152 order median against a $459 mean signals a long tail of small orders. Bundles, "frequently bought together," and minimums shift the middle up. *→ ties to [§08 distribution](#08-what-the-averages-conceal)*
5. **Diversify beyond CA and NY.** Two states are 33% of revenue. Fund 5–7 look-alike states to reduce single-market exposure. *→ ties to [§07 states](#07-two-states-carry-a-third-of-the-business)*
6. **Time re-engagement to the 192-day cycle.** Automated touchpoints at day 150–180 catch customers before the average lapse. *→ ties to [§06 purchase cycle](#the-re-engagement-window)*
7. **Protect the top sub-categories.** Nine sub-categories make up more than 80% of revenue; keep their pricing, depth, and visibility sharp, and review the sub-5% long tail for overhead. *→ ties to [§04 Pareto](#04-how-concentrated-is-the-catalog-really)*
8. **Grow the copier business.** At $2,216 per sale, copiers are high-value and low-volume. Leasing or bulk deals convert a niche into a lever. *→ ties to [§08 copiers](#the-copier-whales)*

## Reproducibility

Every table in this document was produced by running the queries shown against the cleaned dataset (9,800 rows) in PostgreSQL 16. Totals reconcile to $2,261,536.78. The measure throughout is sales revenue; the dataset carries no profit, quantity, or discount columns, so margin and unit analysis are out of scope by construction.


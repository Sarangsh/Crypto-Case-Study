# Crypto-Case-Study

> A SQL Case Study performed on Daily BTC data available on the [Serious SQL](https://www.datawithdanny.com) course by Danny Ma.

## Table of Contents 📖

* [🔭 Dataset Exploration](#explore)
* 🧽 Data Cleaning
* [📊 Business Problem Solutions](#solutions)
-------------------
# 🔭 Dataset Exploration <a name='explore'></a>

The `daily_btc` table consists of 7 rows giving us a detailed view of the BTC values in the market. 

```
SELECT *
FROM trading.daily_btc
LIMIT 3;
```

|market_date|	open_price|	high_price|	low_price|	close_price|	adjusted_close_price|	volume|
|---|---|---|---|---|---|---|
|2014-09-17|	465.864014|	468.174011|	452.421997|	457.334015|	457.334015|	21056800|
|2014-09-18|	456.859985|	456.859985|	413.104004|	424.440002|	424.440002|	34483200|
|2014-09-19|	424.102997|	427.834991|	384.532013|	394.795990|	394.795990|	37919700|

> Understanding the structure and the meaning behind it.

|Column Name| Description|
|---|---|
|market_date|	Cryptocurrency markets trade daily with no holidays|
|open_price|	$ USD price at the beginning of the day|
|high_price|	Intra-day highest sell price in $ USD|
|low_price|	Intra-day lowest sell price in $ USD|
|close_price|	$ USD price at the end of the day|
|adjusted_close_price|	$ USD price after splits and dividend distributions|
|volume|	The daily amount of traded units of cryptocurrency|

# 🧽 Data Cleaning <a name='clean'></a>

### Identifying Null Rows

A simple query to perform in order to retrieve all null value columns from the data is, 

```sql
SELECT *
FROM trading.daily_btc
WHERE
  market_date IS NULL
  OR open_price IS NULL
  OR high_price IS NULL
  OR low_price IS NULL
  OR close_price IS NULL
  OR adjusted_close_price IS NULL
  OR volume IS NULL
);
```
### Updating Null Rows

 `LAG` and `LEAD` both can be used to simultaneously fill out the null values in a table, replacing nulls with the previous record. 
 
 Using `LAG` in combination with `COALESCE` we can create an updated version of the `daily_btc` table that would help us solve the business questions. 
 
 ```SQL
DROP TABLE IF EXISTS updated_daily_btc;
CREATE TEMP TABLE updated_daily_btc AS
SELECT
  market_date,
  COALESCE(
    open_price,
    LAG(open_price, 1) OVER w,
    LAG(open_price, 2) OVER w
  ) AS open_price,
  COALESCE(
    high_price,
    LAG(high_price, 1) OVER w,
    LAG(high_price, 2) OVER w
  ) AS high_price,
  COALESCE(
    low_price,
    LAG(low_price, 1) OVER w,
    LAG(low_price, 2) OVER w
  ) AS low_price,
  COALESCE(
    close_price,
    LAG(close_price, 1) OVER w,
    LAG(close_price, 2) OVER w
  ) AS close_price,
  COALESCE(
    adjusted_close_price,
    LAG(adjusted_close_price, 1) OVER w,
    LAG(adjusted_close_price, 2) OVER w
  ) AS adjusted_close_price,
  COALESCE(
    volume,
    LAG(volume, 1) OVER w,
    LAG(volume, 2) OVER w
  ) AS volume
FROM trading.daily_btc
-- NOTE: checkout the syntax where I've included an unused window below!
WINDOW
  w AS (ORDER BY market_date),
  unused_window AS (ORDER BY market_date DESC);

-- inspect a few rows of the updated dataset for October
SELECT *
FROM updated_daily_btc
WHERE market_date BETWEEN '2020-10-08'::DATE AND '2020-10-13'::DATE;
```
|market_date|	open_price|	high_price|	low_price|	close_price|	adjusted_close_price|	volume|
|---|---|---|---|---|---|---|
|2020-10-08|	10677.625000|	10939.799805|	10569.823242|	10923.627930|	10923.627930|	21,962,121,001|
|2020-10-09|	10677.625000|	10939.799805|	10569.823242|	10923.627930|	10923.627930|	21,962,121,001|
|2020-10-10|	11059.142578|	11442.210938|	11056.940430|	11296.361328|	11296.361328|	22,877,978,588|
|2020-10-11|	11296.082031|	11428.813477|	11288.627930|	11384.181641|	11384.181641|	19,968,627,060|
|2020-10-12|	11296.082031|	11428.813477|	11288.627930|	11384.181641|	11384.181641|	19,968,627,060|
|2020-10-13|	11296.082031|	11428.813477| 11288.627930|	11384.181641|	11384.181641|	19,968,627,060|

# 📊 Business Problem Solutions <a name='solutions'></a>

### Q1: What is the earliest & latest <code>market_date</code> value? 

```sql

CREATE temp table market_dates AS 

SELECT market_date, 

RANK() OVER(
ORDER BY market_date) as Ascending,


RANK() OVER(
ORDER BY market_date desc) as Descending

FROM trading.daily_btc;

select * from market_dates;
```

|Ascending|Descending|
|---|---|
|2014-09-17|2021-02-24|

The earliest date is, 17th of September 2014 and the latest date is, 24th February 2021.

### Q2: What was the historic all-time high and low values for the <code>close_price</code> rounded to 6 decimal places and their dates?

```sql
SELECT close_price, market_date FROM trading.daily_btc where close_price IS NOT NULL ORDER BY close_price DESC LIMIT 1)

UNION

(SELECT close_price, market_date FROM trading.daily_btc ORDER BY close_price ASC LIMIT 1)
```

|Market Date|Close Price|
|---|---|
|2015-01-14|178.102997|
|2021-02-21|57539.945313|

### Q3: Which date had the most <code>volume</code> traded (to nearest integer) and what was the <code>close_price</code> for that day?

```sql
SELECT market_date, volume, ROUND(close_price) AS rounded_price FROM trading.daily_btc 
WHERE volume IS NOT NULL and close_price IS NOT NULL ORDER BY 2 DESC;
```

|Market Date|Volume|Rounded Price|
|---|---|---|
|2021-01-11|123320567399|35567|

### Q4: How many days had a <code>low_price</code> price which was 10% less than the <code>open_price</code> - what percentage of the total number of trading days is this rounded to the nearest integer?

> 💡 Hint: remove days where volume is null

```sql
WITH cte AS (
SELECT
SUM(
CASE
WHEN low_price < 0.9 * open_price THEN 1
ELSE 0
END
) AS low_days,
COUNT(*) AS total_days
FROM trading.daily_btc
WHERE volume IS NOT NULL
)
SELECT
low_days,
ROUND(100 * low_days / total_days) AS _percentage
FROM cte;
```
|Low Days|Percentage|
|---|---|
|79|3%|

### Q5: What percentage of days have a higher `close_price` than `open_price`?

```sql

WITH cte AS (
 SELECT SUM(
 CASE WHEN close_price > open_price THEN 1
 ELSE 0
 END) AS good_days,
 COUNT(*) AS total_days
 FROM trading.daily_btc
)
SELECT good_days, ROUND(100.0 * good_days / total_days) AS percentage FROM cte;
```

|Good Days|Percentage|
|---|---|
|1283|55%|

> 💡 Point To Remember: Remember integer floor dvision!

### Q6: What was the largest difference between <code>high_price</code> and <code>low_price</code> and which date did it occur? 

```sql
select (high_price - low_price) AS difference, market_date from trading.daily_btc ORDER BY difference DESC NULLS LAST LIMIT 1;
```

|Difference|Market Date|
|---|---|
|8914.339844|2021-02-23|

### Q7: If you invested $10,000 on the 1st January 2016 - how much is your investment worth in 1st of January 2021 and what is your total growth of your investment as a percentage of your original investment? Use the `close_price` for this calculation.

```sql
WITH start_investment AS (
SELECT
10000 / close_price AS btc_volume,
close_price AS start_price
FROM trading.daily_btc
WHERE market_date = '2016-01-01'
),
end_investment AS (
SELECT
close_price AS end_price
FROM trading.daily_btc
WHERE market_date = '2021-01-01'
)
SELECT
btc_volume,
start_price,
end_price,
ROUND(100 * (end_price - start_price) / start_price) AS growth_rate,
ROUND(btc_volume * end_price) AS final_investment
FROM start_investment
CROSS JOIN end_investment;
```

|BTC Volume|Start Price|End Price|Growth Rate|Final Investment|
|---|---|---|---|---|
|23.0237551162093533|434.334015|29374.152344|6663|676303|

----------------------

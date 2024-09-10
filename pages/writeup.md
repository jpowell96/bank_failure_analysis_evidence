# FDIC Bank Failures - Time Series Analysis

My latest endeavor with SQL is working with timeseries data.
For this mini-analysis, I decided to chart out US bank failures overtime.

I found an [official dataset of FDIC Bank Failures](https://catalog.data.gov/dataset/fdic-failed-bank-list) on data.gov, which has public
datasets from the US governemnt. It lists all the banks that have failed since 2000.

## Importing the Data

To import the data into my Postgres Database I did the following steps
1. Create a table to represent the rows in the CSV file
```sql
CREATE TABLE bank_failures (
id bigserial,
bank_name TEXT,
city TEXT,
state TEXT,
cert TEXT,
acquiring_institution TEXT,
closing_date DATE,
fund BIGINT
);
```

2. Copy the data into the table from the csv file, using the [COPY command](https://www.postgresql.org/docs/current/sql-copy.html)
```sql
COPY bank_failures (
bank_name,
city,
state,
cert,
acquiring_institution,
closing_date,
fund
) FROM '/path/to/banklist.csv' WITH (FORMAT csv, DELIMITER ',', QUOTE '"', ESCAPE '"', ENCODING 'UTF-8');
```

I originally set the ENCODING to ASCII, but after running into some issues realized I needed to set the encoding to 'UTF-8'. 
Now, with the correct character encoding, 
I imported my data for real but got this error message: 

`ERROR:  invalid byte sequence for encoding "UTF8": 0x96`

After some digging online, I realized this meant there was a non UTF-8 character in the csv file. There was a hyphen character '-' for one of the banks was a special character. I replaced it with a "regular" hyphen - character and that allowed me to import all the data.

Once imported, the data looked like this:

|id|bank_name|city|state|cert|acquiring_institution|closing_date|fund|
|--|---------|----|-----|----|---------------------|------------|----|
|40|Citizens Bank|Sac City|IA|8758|Iowa Trust & Savings Bank|2023-11-03|10545|
|41|Heartland Tri-State Bank|Elkhart|KS|25851|Dream First Bank, N.A.|2023-07-28|10544|
|42|First Republic Bank|San Francisco|CA|59017|JPMorgan Chase Bank, N.A.|2023-05-01|10543|
|43|Signature Bank|New York|NY|57053|Flagstar Bank, N.A.|2023-03-12|10540|
|44|Silicon Valley Bank|Santa Clara|CA|24735|First-Citizens Bank & Trust Company|2023-03-10|10539|


## Querying the data

My goal was to run a few simple queries to get the number of bank failures by year,
and then another query to get all the bank failures by month.

I started with a query to find all the months/years that banks had failures

```sql bank_failures_by_month_year
-- Return bank failures by month, year based on the dataset
 SELECT
  EXTRACT('YEAR' FROM closing_date :: DATE) as closing_year, 
  EXTRACT('MONTH' FROM closing_date :: DATE) as closing_month, 
  COUNT(*) as total_failures  
FROM bank_failures_db.all_bank_failures 
GROUP BY closing_year, closing_month
ORDER BY closing_year, closing_month;
```

<DataTable data={bank_failures_by_month_year}/>

To my surprise, there were some months missing from this query. There wasn't a row for months from January - October, 2000. 

It may seem odd, but this is the expected output. If there weren't any failures for a given month in a year, there wouldn't be rows. If I want there to be rows for every month from 2000 - 2023, I need to have date for those months too.

To accomplish this I made a table called timescale, that contains all the months, years from January, 2000 - December 2023. My initial attempt was overcomplicated, but I realized PostgreSQL provides a helpful function, [generate_series()](https://www.postgresql.org/docs/current/functions-srf.html) That lets me generate a series of rows between to dates:

```sql
-- Generate a row for every month between Jan, 2000 and Dec, 2023
SELECT generate_series('2000-01-01', '2023-12-01','1 month'::interval) as timescale;
```

```sql
-- Generate a row for every year between Jan, 2000 and Dec, 2023
SELECT generate_series('2000-01-01', '2023-12-01','1 year'::interval) as timescale;
```

This code will return a result set containing a row for every month from January 2000 until December 2023. 

With this table in mind, we can redo our original "Failures per Month" query and JOIN the timescale onto our bank_failures table:


```sql bank_failures_by_month_year_with_time_scale
SELECT
  date_trunc('month', bank_failures_db.timescale.year_month) as closing_month,
  COUNT(bank_failures_db.all_bank_failures.closing_date) as total_failures  
-- the series table has complete dates, so join bank_failures onto the series for complete data for the complete time period
FROM bank_failures_db.timescale LEFT JOIN bank_failures_db.all_bank_failures
ON date_trunc('month', bank_failures_db.timescale.year_month) = date_trunc('month', bank_failures_db.all_bank_failures.closing_date)
GROUP BY closing_month
ORDER BY closing_month; 
```
<BarChart 
 data={bank_failures_by_month_year_with_time_scale}
 x=closing_month
 y=total_failures
 title="Failures Over Time"
/>

Important Notes:
1. The bank_failures table is left joined onto the timescale. This is done because the timescale
includes every month, year while the bank_failures table only includes records for month/years where a bank failed.
2. We do a `COUNT(bank_failures.closing_date)` rather than a  `COUNT(*)`
    -   A left join will return a record for each row on the timescale table. COUNT(*) would 
    return 1 failure, because it counts rows including NULLs. `COUNT(bank_failures.closing_date)` ensures that we only count cases where there were bank failure in a given month.

Likewise, if we wanted to upsample the data to yearly buckets, the query would look like this:

```sql
with series as (SELECT
generate_series('2000-01-01', '2023-12-01','1 year'::interval) as timescale)
SELECT
  EXTRACT('YEAR' FROM timescale) as closing_year, 
  COUNT(bank_failures.closing_date) as total_failures  
FROM series LEFT JOIN bank_failures 
on EXTRACT('YEAR' from timescale) = EXTRACT('YEAR' from closing_date :: DATE)
GROUP BY closing_year
ORDER BY closing_year;
```

We've updated the generate_series() function to use an interval of 1 year rather than 1 month, and
updated our JOIN condition to be based on matching years.


## Analyzing the Data


Here are some of the queries I ran on the dataset:

How many bank closed over the entire time period:
```sql total_bank_failures
select count(*) as failures from bank_failures_db.all_bank_failures  bf ;
```
<DataTable data={total_bank_failures}/>

Which states had the most bank failures over the complete time period?

```sql failures_by_state_top_10
select 
    state, 
    COUNT(*) as bank_failures
from bank_failures_db.all_bank_failures bf 
group by state
order by COUNT(*) desc
limit 10;
```

<BarChart data={failures_by_state_top_10}
  x=state
  y=bank_failures
  swapXY=true
/>


For each year, which state(s) had the most bank failures? Include ties.

```sql
with ranked_state_failures_by_year as (
	select 
	 extract('YEAR' from closing_date) as closing_year,
	 state,
	 COUNT(*) as state_failures_by_year,
	 -- Window functions are evaluated AFTER the group by clause, so this is comparing the bank failures per state
	 dense_rank() over (partition by extract('YEAR' from closing_date) order by  COUNT(*) desc) 
	from bank_failures bf 
	group by extract('YEAR' from closing_date), state
	order by extract('YEAR' from closing_date), COUNT(*) desc),
all_years as (
	select generate_series('01-01-2000', '01-01-2023', '1 year'::interval) as timescale
)
select 
    extract('YEAR' from all_years.timescale) as closing_year, 
    coalesce(state, 'N/A') as state, 
    coalesce(state_failures_by_year, 0) as state_failures_by_year
from 
all_years left JOIN
ranked_state_failures_by_year
on  extract('YEAR' from all_years.timescale) = ranked_state_failures_by_year.closing_year 
-- Some years do not have bank failures, so we include a null check to ensure they appear in the result set
where dense_rank = 1 or dense_rank is null
order by all_years.timescale;
``` 

```sql timescale_years
select EXTRACT('year' from Year) as Year FROM bank_failures_db.timescale_year order by Year;
```
<Dropdown
    data={timescale_years} 
    name=timescale_year_dropdown
    value=Year
/>

```sql ranked_state_failures_by_year
with ranked_state_failures_by_year as (
	select 
	 extract('YEAR' from closing_date) as closing_year,
	 state,
	 COUNT(*) as state_failures_by_year,
	 -- Window functions are evaluated AFTER the group by clause, so this is comparing the bank failures per state
	 dense_rank() over (partition by extract('YEAR' from closing_date) order by  COUNT(*) desc) as dense_rank
	from bank_failures_db.all_bank_failures bf 
	where closing_year = '${inputs.timescale_year_dropdown.value}'
	group by extract('YEAR' from closing_date), state
	order by extract('YEAR' from closing_date), COUNT(*) desc)
select 
    closing_year, 
    coalesce(state, 'N/A') as state, 
    coalesce(state_failures_by_year, 0) as state_failures_by_year,
	dense_rank
from ranked_state_failures_by_year
order by dense_rank, ranked_state_failures_by_year.closing_year;
```

<DataTable data={ranked_state_failures_by_year}/>

```sql all_states
select distinct state as state
from bank_failures_db.all_bank_failures
order by state;
```

```sql failures_by_year_per_state
select 
bank_failures_db.timescale_year.year,
COUNT(subquery.id) as bank_failures
from
bank_failures_db.timescale_year LEFT JOIN 
(SELECT * FROM bank_failures_db.all_bank_failures where  bank_failures_db.all_bank_failures.state = '${inputs.all_states_dropdown.value}') as subquery
on date_trunc('year',bank_failures_db.timescale_year.year) = date_trunc('year',subquery.closing_date)

group by bank_failures_db.timescale_year.year
order by bank_failures_db.timescale_year.year;
```

<Dropdown
    data={all_states} 
    name=all_states_dropdown
    value=state
/>
<BarChart
  data={failures_by_year_per_state}
  x=year
  y=bank_failures
/>
<DataTable data={failures_by_year_per_state}/>


Which years had the most bank failures?
```sql
select 
	RANK() over (order by sum(total_failures) desc),
	closing_year::int, 
	SUM(total_failures) as total_failures
from bank_failures_by_month 
group by closing_year
order by SUM(total_failures) desc
limit 5;
```

|rank|closing_year|total_failures|
|----|------------|--------------|
|1|2010|157|
|2|2009|140|
|3|2011|92|
|4|2012|51|
|5|2008|25|


Which month/year had the most bank failures?

```sql
select 
	RANK() over (order by total_failures desc),
	closing_year :: INT, 
	closing_month,
	total_failures
from bank_failures_by_month 
order by total_failures desc
limit 5;
```

|rank|closing_year|closing_month|total_failures|
|----|------------|-------------|--------------|
|1|2009|7|24|
|2|2010|4|23|
|3|2010|7|22|
|4|2009|10|20|
|5|2010|3|19|


What was percent change each year?

```sql
with bank_failures_by_year as (
-- Return a table with the year, total_failures, and toal failures in the previous year, defaulting to 0 if not data is found
SELECT
  EXTRACT('YEAR' FROM timescale) as closing_year, 
  COUNT(bank_failures.closing_date) as total_failures,
  coalesce(LAG(COUNT(bank_failures.closing_date)) over (order by EXTRACT('YEAR' FROM timescale)),0) as prev_year_failures
FROM (SELECT
generate_series('2000-01-01', '2023-12-01','1 year'::interval) as timescale) as series LEFT JOIN bank_failures 
-- Join the series of years with the bank failures table based on matching years
on EXTRACT('YEAR' from timescale) = EXTRACT('YEAR' from closing_date)
GROUP BY closing_year
ORDER BY closing_year)
-- Calculate the year over year change between each year of bank failures
select closing_year, total_failures, 
	case 
		when closing_year = 2000 then 0 || '%'
		when prev_year_failures = 0 then total_failures * 100 || '%' 
		else ROUND(((total_failures - prev_year_failures) :: NUMERIC / prev_year_failures) * 100.0, 2) || '%' 
	end as yoy_pct_change
from bank_failures_by_year;
```

|closing_year|total_failures|yoy_pct_change|
|------------|--------------|--------------|
|2006|0|0%|
|2007|3|300%|
|2008|25|733.33%|
|2009|140|460.00%|
|2010|157|12.14%|
|2011|92|-41.40%|
|2012|51|-44.57%|
|2013|24|-52.94%|
|2014|18|-25.00%|
|2015|8|-55.56%|
|2016|5|-37.50%

## Readout

Over the time period from 2000 to 2023, there were 568 banks that closed. Georiga, Florida, and Illinois had the most closers over the time period with 93, 76, and 69 closures respectively. On a year basis, Florida had the most closures in a single year with 29 banks closing in 2010.

As expected, bank closures treneded upwards between 2008 - 2012, peaking with 157 closures in 2010. The year over year changes highlight this explosion of failures.

From 2007 to 2008, the number of bank closures grew over 8x from 3 closures to 25 closures, and grew even more the following year - from 25 closures in 2008 to 140 in 2009 - a 460% increase. Closures in a single year peaked in 2010 with 157 closures with Florida having the most closures with 29 bank clousres. July 2009 was the worst month for closures, with 29 closings in a single month.


## Wrapping it up

For time series data that has gaps, the generate_series() function can enable 
analyzing the data with a complete timeline. And, if needed, can support different
levels of granularity (day, week, month, etc.) to match the granularity of your date.

## Improvements

1. I made use of the EXTRACT function often in my queries. It might make sense to just have columns for month, year since I was using extract so often. 

2. Each row has an acquiring institution. It would be interesting to revisit
the dataset and use some recursive queries to uncover chains of acquisitions.

3. Normalizing the some columns such as state, acquiring institution, institution could make it easier to run queries related to who acquired which banks. FDIC has additional datasets that assign a unique id to institutions. An enhancement to this project would be to import all FDIC institutions and use that as a reference to normalize data for institutions in the bank failures dataset.

## Helpful Links

1. [FDIC Bank Failures Homepage](https://www.fdic.gov/resources/resolutions/bank-failures/)

2. [generate_series() postgres docs](https://www.postgresql.org/docs/current/functions-srf.html)

3. [timescale db - generate series explanation](https://www.timescale.com/blog/how-to-create-lots-of-sample-time-series-data-with-postgresql-generate_series/)

4. [Evidence](https://docs.evidence.dev/)
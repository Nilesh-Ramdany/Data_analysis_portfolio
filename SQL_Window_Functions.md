# SQL Window Functions

The aim of this project is to use window functions in SQL statements to understand dow they work and their use cases.

First I will create a view which I will use to run the queries against.

````sql
create or replace view sales_data as
select date(s.SaleDate) as date, p.salesperson, g.region, sum(s.amount) as amount
from sales as s
join geo as g on g.GeoID = s.GeoID
join people p on p.SPID = s.SPID
group by Salesperson, g.region, SaleDate;
````
Here's how the data looks like:

````sql
Select  * 
From sales_data
LIMIT 5;
````
| date      | salesperson      | region      | amount      |
|---------| ---------------- | ----------- |-----------|
| 2021-01-01 | Barr Faughny     | APAC        | 8673        |
| 2021-01-01 | Dennison Crosswaite | Americas    | 532         |
| 2021-01-01 | Karlen McCaffrey | Americas    | 8428        |
| 2021-01-01 | Beverie Moffet   | Americas    | 28735       |
| 2021-01-01 | Rafaelita Blaksland | APAC        | 6692        |

## Running Total

Get the running total for the sales amount

````sql
Select
  date, 
  SUM(amount) as sales_amount,
  SUM(SUM(amount)) OVER (order by date) as running_total
FROM sales_data
GROUP BY date
LIMIT 5;
````
| date      | sales_amount      | running_total      |
| --------- | -----------------| ------------------ |
| 2021-01-01 | 167664            | 167664             |
| 2021-01-04 | 69916             | 237580             |
| 2021-01-05 | 123753            | 361333             |
| 2021-01-06 | 94045             | 455378             |
| 2021-01-07 | 54306             | 509684             |

## Ranking Sales Person by Sales Amount

I have rounded down the sales amount so that we can see the difference between the ranking functions.

````sql
SELECT
  salesperson,
  Round(Sum(amount), -4) as sales,
  row_number() OVER (ORDER BY round(sum(amount), -4)) as rownumber,
  rank() OVER (ORDER BY round(sum(amount), -4)) as sales_rank,
  dense_rank() OVER (ORDER BY round(sum(amount), -4)) as sales_denserank
FROM sales_data
GROUP BY salesperson
LIMIT 7;
````

| salesperson      | sales      | rownumber      | sales_rank      | sales_denserank      |
| ----------------|---------- | -------------- | --------------- | -------------------- |
| Jan Morforth     | 1580000    | 1              | 1               | 1                    |
| Brien Boise      | 1600000    | 2              | 2               | 2                    |
| Curtice Advani   | 1610000    | 3              | 3               | 3                    |
| Husein Augar     | 1610000    | 4              | 3               | 3                    |
| Kaine Padly      | 1630000    | 5              | 5               | 4                    |
| Mallorie Waber   | 1640000    | 6              | 6               | 5                    |
| Andria Kimpton   | 1670000    | 7              | 7               | 6                    |

- **row_number()** - unique number for each row within partition, with different numbers for tied values.
- **rank()** - ranking within partition with gaps and same ranking for tied values.
- **dense_rank()** - ranking within partition, with no gaps and same ranking for tied values. 

We can use a **named window definition** to make the queries with window functions cleaner cleaner,

````sql
Select
  salesperson,
  region,
  round(sum(amount), -4) as sales,
  row_number() over w as rownumber,
  rank() over w as sales_rank,
  dense_rank() over w as sales_denserank
From sales_data
Group By salesperson, region
window w as(
	partition by region
	order by round(sum(amount), -4));
````

## Top 3 Salesperson in each Region

````sql
Select * 
From(
  Select
    salesperson, 
    region,
    sum(amount) as sales_amount,
    rank() over (partition by region order by sum(amount)) as rnk
	From sales_data
	Group By salesperson, region
    ) x 
Where x.rnk < 4;
````
| salesperson      | region      | sales_amount      | rnk      |
| ---------------- | ----------- | ----------------- | -------- |
| Andria Kimpton   | Americas    | 489860            | 1        |
| Curtice Advani   | Americas    | 506457            | 2        |
| Husein Augar     | Americas    | 516677            | 3        |
| Camilla Castle   | APAC        | 768439            | 1        |
| Mallorie Waber   | APAC        | 800177            | 2        |
| Gigi Bohling     | APAC        | 816662            | 3        |
| Barr Faughny     | Europe      | 208530            | 1        |
| Beverie Moffet   | Europe      | 217140            | 2        |
| Brien Boise      | Europe      | 222838            | 3        |

## Lead and Lag

````sql
Select
  month(date) as month,
  sum(amount) as sales_amount,
  lag(sum(amount)) over (order by month(date)) as prev_month_sales,
  lead(sum(amount)) over (order by month(date)) as next_month_sales
From sales_data
Group By month(date)
Limit 5;
````
| month      | sales_amount      | prev_month_sales      | next_month_sales      |
| ---------- | ----------------- | --------------------- | --------------------- |
| 1          | 8238111           |          NULL             | 5038369               |
| 2          | 5038369           | 8238111               | 5501405               |
| 3          | 5501405           | 5038369               | 2885253               |
| 4          | 2885253           | 5501405               | 2794491               |
| 5          | 2794491           | 2885253               | 2626162               |

## Query to Show Increase or Decrease

````sql
Select
  month(date) as month,
  sum(amount) as sales_amount,
  case when sum(amount) < lag(sum(amount)) OVER w then 'decrease'
    when sum(amount) > lag(sum(amount)) OVER w then 'increase'
    when sum(amount) = lag(sum(amount)) OVER w then 'same'
    else null
  end as change_in_sales
From sales_data
Group By month(date)
window w as (
  Order By month(date))
Limit 5;
````
| month      | sales_amount      | change_in_sales      |
| ---------- | ----------------- | -------------------- |
| 1          | 8238111           |         NULL             |
| 2          | 5038369           | decrease             |
| 3          | 5501405           | increase             |
| 4          | 2885253           | decrease             |
| 5          | 2794491           | decrease             |



## Amount Sold on the First and Last sale date of each month

````sql
With CTE as(
  Select
    date,
    first_value(sum(amount)) OVER (Partition by month(date) order by date) as sales_first_day,
    last_value(sum(amount))
      OVER (  Partition by month(date) order by date
              Range Between Unbounded Preceding
              and Unbounded Following) as sales_last_day,
    rank() over (partition by month(date) order by date) as rnk
  From sales_data
  Group by date
  Order by date
  )

Select
  month(date) as month,
  sales_first_day,
  sales_last_day
From CTE
-- want only one output for each month
Where rnk < 2
Limit 5;
````
| month     | sales_first_day      | sales_last_day      |
| ---------------- | -------------------- | ------------------- |
| 1                | 167664               | 138523              |
| 2                | 182427               | 120316              |
| 3                | 119819               | 7119                |
| 4                | 110292               | 112581              |
| 5                | 196196               | 184366              |

## Distribution Functions

- **percent_rank()** - the percentile ranking number of a row—a value in [0, 1] interval
- **cume_dist()** - the cumulative distribution of a value within a group of values, i.e., the number of rows with values less than or equal to the current row’s value divided by the total number of rows; a value in (0, 1] interval

````sql
Select
  salesperson,
  sum(amount) as sales_amount,
  round(percent_rank() over (order by sum(amount)), 2) as percent_rnk,
  round(cume_dist() over (order by sum(amount)),2) as cume_distr
From sales_data
Group by salesperson;
````
| salesperson      | sales_amount      | percent_rnk      | cume_distr      |
| ---------------- | ----------------- | ---------------- | --------------- |
| Jan Morforth     | 1584975           | 0                | 0.04            |
| Brien Boise      | 1602559           | 0.04             | 0.08            |
| Curtice Advani   | 1607249           | 0.08             | 0.12            |
| Husein Augar     | 1612394           | 0.12             | 0.16            |
| Kaine Padly      | 1631707           | 0.17             | 0.2             |
| Mallorie Waber   | 1642816           | 0.21             | 0.24            |
| Andria Kimpton   | 1674302           | 0.25             | 0.28            |
| Marney O'Breen   | 1678971           | 0.29             | 0.32            |
| Gigi Bohling     | 1711066           | 0.33             | 0.36            |
| Beverie Moffet   | 1714888           | 0.38             | 0.4             |
| Camilla Castle   | 1726746           | 0.42             | 0.44            |
| Jehu Rudeforth   | 1734047           | 0.46             | 0.48            |
| Oby Sorrel       | 1740368           | 0.5              | 0.52            |
| Rafaelita Blaksland | 1750105           | 0.54             | 0.56            |
| Kelci Walkden    | 1775473           | 0.58             | 0.6             |
| Ches Bonnell     | 1780660           | 0.62             | 0.64            |
| Dotty Strutley   | 1785091           | 0.67             | 0.68            |
| Barr Faughny     | 1794940           | 0.71             | 0.72            |
| Van Tuxwell      | 1801023           | 0.75             | 0.76            |
| Dennison Crosswaite | 1802150           | 0.79             | 0.8             |
| Karlen McCaffrey | 1844955           | 0.83             | 0.84            |
| Madelene Upcott  | 1859305           | 0.88             | 0.88            |
| Roddy Speechley  | 1892163           | 0.92             | 0.92            |
| Gunar Cockshoot  | 1899436           | 0.96             | 0.96            |
| Wilone O'Kielt   | 1914157           | 1                | 1               |

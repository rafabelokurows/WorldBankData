# Queries
Queries used to obtain insight into indicators, comparing the state of education in Portugal against the entire European continent and against itself a few years ago.

# Comparison Portugal 2000 x 2014

```
select 
    LEFT(i.name, position('(' in i.name) - 1) as name,
    country.ind2000,
    continent.ind2014,
    (continent.ind2014/(country.ind2000)-1)*100 as change
from(  
    select indicator_id,value as ind2000 from education_data where 
    indicator_id in(64,151,1,107)  and
        country_id = 258 and extract(year from year) = 2000) as country 
    full outer join
    (
    select indicator_id,value as ind2014 from education_data where 
    indicator_id in(64,151,1,107) and
        country_id = 258 and extract(year from year) = 2014) as continent on country.indicator_id = continent.indicator_id
    join indicators i on country.indicator_id = i.id
order by
    name
```

# Portugal x Europe - 2ary ed.
```
select 
    LEFT(i.name, position('(' in i.name) - 1) as name,
    country.portugal,
    continent.euro
from(  
    select indicator_id,value as portugal from education_data where indicator_id in(82,131,135,107)  and
        country_id = 193 and extract(year from year) = 2014) as country 
    full outer join
    (
    select indicator_id,value as euro from education_data where indicator_id in(82,131,135,107) and
        country_id = 67 and extract(year from year) = 2014) as continent on country.indicator_id = continent.indicator_id
    join indicators i on country.indicator_id = i.id
 ```

  # Correlation between enrollment and expenditure
 ``` 
  select corr(foo.male,foo.female) as cor from(

select 
    male.year,
    male.male,
    female.female
    
from(  
    select extract(year from year) as year,value as male from education_data where indicator_id = 82 and
        country_id = 193) as male 
    full outer join
    (
    select extract(year from year) as year,value as female from education_data where indicator_id = 131 and
        country_id = 193) as female on male.year = female.year
where
    male is not null and female is not null
  ) as foo  
  ```

  # Buckets - % of investment in education
  
  ```
  select 1 + ((bucket-1) * (8-1)/5) || '-' || 1 + ((bucket) * (8-1)/5)|| '%' as investment, 
       cnt as countries from (
select 
width_bucket(value, 1,8,5) as bucket,count(*) as cnt 
from education_data ed where ed.indicator_id = 78 and extract(YEAR from ed.year) = 2014
group by bucket) x
```

# Enrollment in 3ary ed.
```
select 
    indicator_id,
    value,
    year
from 
    education_data 
where 
    indicator_id in(93) and--School Enrollment, Tertiary
    country_id = 258 and--World
    extract(year from year)in(2000,2010,2013,2014)
order by
    year asc
    ```

# Enrollment x Expenditure - scatter plot
```
select 
    expd.year,
    expd.expd,
    enroll.enroll
    
from(  
    select extract(year from year) as year,value as expd from education_data where indicator_id = 82 and
        country_id = 193) as expd 
    full outer join
    (
    select extract(year from year) as year,value as enroll from education_data where indicator_id = 131 and
        country_id = 193) as enroll on expd.year = enroll.year
where
    expd is not null and enroll is not null
    ```

# Expenditure on education per level in Portugal
```
select
    indicator_id,
    country_id,
    value,
    name,
    year 
from
    education_data ed 
    join countries c on ed.country_id = c.id 
where
    c.is_country = true and 
    ed.indicator_id in(80,82,84) and 
    country_id = 193
```

# Investment in Primary Ed x Bachelor's degrees 12 years later
```
select 
    expd.country_id,
    expd.expd,
    enroll.enroll
    
from(  
    select country_id,extract(year from year) as year,value as expd from education_data where indicator_id = 85 and extract(YEAR from year) = 2002 ) as expd 
    full outer join
    (
    select country_id,extract(year from year) as year,value as enroll from education_data where indicator_id = 104 and
        extract(YEAR from year) = 2014) as enroll on  expd.country_id = enroll.country_id
where
    expd is not null and enroll is not null
```

# Is the portuguese people aging?
```
select 
    adults.year,
    adults.adults,
    kids.kids,
    100-adults-kids as seniors
from(  
    select extract(year from year) as year,value as adults from education_data where indicator_id =71 and
        country_id = 193) as adults 
    full outer join
    (
    select extract(year from year) as year,value as kids from education_data where indicator_id =72 and
        country_id = 193) as kids on adults.year = kids.year

```

# Which country hast the most complete indicators

```
/*Q4 Which actual country* has the most complete data? Use the definition for most complete
as the country with the highest median number of observations per measured indicator.
Please specify a query that returns exactly one row with a “country_name” and
“median” columns.*/
select 
    foo.country_name as country_name,
    percentile_disc(0.5) within group (order by foo.cont) as median 
from (
    select 
        count(*) as cont,
        country_id,
        c.name as country_name,
        indicator_id
    from
        education_data join countries c on education_data.country_id = c.id 
    where
        is_country = true 
    group by
        country_id, country_name, indicator_id 
    order by 
        cont 
     ) as foo 
group by
    foo.country_id,foo.country_name 
order by median desc limit 1
--this means that Morocco had the largest median for number of observations per measured indicator, a median of 40 observations per each of its 130 measured indicators
```


# Greatest percentage change in an indicator
```
/*Q5 What is the name of the actual country* and the year of observation in which the largest
change has happened in percentage terms in the values of an indicator compared to the
previous observation? Also, include the value for this percentage change and the name
of the indicator in your answer. Only look at countries and indicators where the year
difference between observations is not more than 3 years. Please specify a query that
returns exactly one row with a “country_name”, “indicator_name”, “base_year”
and “percentage_change” columns.*/
select 
    foo.country_name,
    foo.indicator_name,
    foo.base_year,
    foo.percentage_change
        from (
select 
    c.name as country_name,
    i.name as indicator_name,
    ed.indicator_id,
    extract (year from ed.year) as base_year,
    lag(extract (year from ed.year),1) OVER (
            PARTITION BY indicator_id,c.id
            ORDER BY extract (year from ed.year)
        ) as last_year,
    extract (year from ed.year) - lag(extract (year from ed.year),1) OVER (
            PARTITION BY indicator_id,c.id
            ORDER BY extract (year from ed.year)
        ) as diff_years,
    ed.value,
    ed.value - lag(ed.value,1) OVER (
            PARTITION BY indicator_id,c.id
            ORDER BY extract (year from ed.year)
        ) as change,
    (ed.value /(lag(ed.value,1) OVER (
            PARTITION BY indicator_id,c.id
            ORDER BY extract (year from ed.year)
        ))-1)*100   as percentage_change
from
    education_data ed 
    join countries c on c.id = ed.country_id 
    join indicators i on ed.indicator_id = i.id
where 
    c.is_country = true and
    ed.value <> 0
order BY
    percentage_change desc
) as foo
where foo.diff_years <= 3
order by abs(percentage_change) desc
limit 1
--most of the largest changes were from countries that had tiny percentages in one year and then huge percentages on the following year
--I excluded indicators that were = 0, but in the real world I guess I would need to perform a deeper analysis before discarding those
```

# Years with the most and the least observations
```
select count(*),extract(YEAR from ed.year)
from education_data ed
where year in((select year from education_data ed group by year order by count(*) desc limit 1),(select year from education_data ed group by year order by count(*) asc,year limit 1))
group by year 
```

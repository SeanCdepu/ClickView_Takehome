# Clickview-Takehome
## Task 1
### A)
- Inner joins – retrieving rows that have matching values in both tables. You would use this when you want to join data from 2 tables and only want the data which is contained in both tables

- Full Outer Join – retrieving rows from both tables if there is a match in either table. You would need this kind of join when you want to link records that are shared across both tables and keep the rest of the data from both tables (data should be relatively small cause it can be costly)

- Left Join – retrieving rows from all of the left table, only retrieving records that match to the left table from the right. This is the most common one, you usually use this to add on extra fields from another table with a linking key

- Right join – same as left join but visa versa. I would usually just use left joins, but you could use right joins for the same purpose as above

- Cross join – creates records of a combination of each record in the left table with each row of the right. These are extremely expensive to use so I don’t do it often but when I do it’s usually to join a single cell of data a new row to my table. E.G
```
Select * from clickview.accounts
cross join
(select company_name from clickview.names limit 1)
This would add a column where every cell is whatever came from clickview_names
```
### B)
Natual joins are like inner joins but will join based on columns with the same name and wont duplicate columns in a select statement e.g. col1, col2, col3 rather than a.col1, a.col2, b.col1, b.col3

## Task 2
--Create a table from the JSON first
```select * from
create or replace table clickview_test
(
  src variant
)
as
select parse_json(column1) as src
from values
('{
   "id": 1,
   "folders": [
{
      "id": 10,
      "name": "History"
},
         {
   "id": 11,
   "name":"Science"
   }
   ]
}') v;

-- Get what I need from it
select * from clickview_test where src:folders.id = '10';
```

## Task 3

Around half of the total micro-partitions are not constant, these would benefit from a re-clustering. The average overlap is somewhat high but not insanely as well as the depth, the higher these are the worse the performance will be. From the histogram you can see that the distribution of the micro-partitions isn’t good, a large portion are in a single bucket (index 1) and the rest of the distribution heavily skews towards the buckets 8-15 which will lead to poor performance as the load is centered on those buckets. I'd recommend re-clustering.

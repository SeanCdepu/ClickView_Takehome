# ClickView-Takehome
## Task 1
### A)
- Inner joins – A join for only retrieving records that has a criterion that exists in both tables. This could be used to find a list of customers that use both the playlist feature AND has watched Romeo & Juliet (assuming these events are in separate tables)

- Full Outer Join – A join for retrieving the full dataset from both tables, linking records that satisfy the join criteria into a single record. This could be used to create a matrix of customer ids based on what events they have and haven’t triggered by outer joining on cust_id in event tables and setting all nulls to 0.
e.g.
|----------| ev1 | ev2 | ev3 | ev4 |
|----------|-----|-----|-----|-----|
| cust_id1 | 1   | 1   | 0   | 1   |
| cust_id2 | 1   | 0   | 1   | 1   |
| cust_id3 | 1   | 0   | 1   | 0   |


- Left Join – The most common join. Adding data from the 2nd table to the 1st, only retrieving data from the 2nd that satisfies the join criteria. This could be used to add columns from a lookup table or add some extra data from another object with a linking id key such as adding billing info to a list of accounts.

- Right join – same as left join but visa versa. I would usually just use left joins, but you could use right joins for the same purpose as above

- Cross join – creates records of a combination of each record in the 1st table with each row of the 2nd. These are extremely expensive to use so I don’t do it often, but they are useful for create grids of data to use as dimension templates such as creating a table for every combination of flavours and sizes of coke
```
select
    flavour,
    size
from
    flavours  CROSS JOIN sizes
```

### B)
Natual joins are like inner joins but will join based on columns with the same name and wont duplicate columns in a select statement e.g. col1, col2, col3 rather than a.col1, a.col2, b.col1, b.col3

## Task 2
```--Create a table from the JSON first
create or replace table scar_playground.clickview_task2
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
],
"libraries": [
    {
    "id": 12,
    "name": "Primary AU"
    },
    {
    "id": 13,
    "name": "Secondary AU"
    }
],
"series":
    {
    "id": 14,
    "name": "Einstein"
    }
}') v;

-- flatten the JSON by folders to access folder ids and data
select 'folders' as key, value:id::number as id, value:name::string as name from scar_playground.clickview_task2
    ,lateral flatten( input => src:folders)
where id = '10';

-- flatten the JSON by libraries to access library ids and data
select 'libraries' as key, value:id::number as id, value:name::string as name from scar_playground.clickview_task2
    ,lateral flatten( input => src:libraries)
where id = '12'
union
-- Can flatten specifically to src since series only has 1 record and contains a key (would need to edit if series data expands)
select key, value:id::number as id, value:name::string as name from scar_playground.clickview_task2
    ,lateral flatten( input => src)
where key = 'series' and id = '14';```
```

## Task 3


## Task 4

Around half of the total micro-partitions are not constant, these would benefit from a re-clustering. The average overlap is somewhat high but not insanely as well as the depth, the higher these are the worse the performance will be. From the histogram you can see that the distribution of the micro-partitions isn’t good, a large portion are in a single bucket (index 1) and the rest of the distribution heavily skews towards the buckets 8-15 which will lead to poor performance as the load is centered on those buckets. I'd recommend re-clustering.

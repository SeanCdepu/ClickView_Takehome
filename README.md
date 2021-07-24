# ClickView-Takehome
## Task 1
### A)
- Inner joins – A join for only retrieving records that has a criterion that exists in both tables. This could be used to find a list of customers that use both the playlist feature AND has watched Romeo & Juliet (assuming these events are in separate tables)

Credit: w3schools aka my heroes (https://www.w3schools.com/sql/img_innerjoin.gif)

- Full Outer Join – A join for retrieving the full dataset from both tables, linking records that satisfy the join criteria into a single record. This could be used to create a matrix of customer ids based on what events they have and haven’t triggered by outer joining on cust_id in event tables and setting all nulls to 0.
example (https://i.imgur.com/j9sXPUK.png)

Credit: w3schools (https://www.w3schools.com/sql/img_fulljoin.gif)

- Left Join – The most common join. Adding data from the 2nd table to the 1st, only retrieving data from the 2nd that satisfies the join criteria. This could be used to add columns from a lookup table or add some extra data from another object with a linking id key such as adding billing info to a list of accounts.

Credit: w3schools (https://www.w3schools.com/sql/img_leftjoin.gif)

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
```
--Create a table from the JSON first
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
```
## A)
```
-- flatten the JSON by folders to access folder ids and data
select 'folders' as key, value:id::number as id, value:name::string as name from scar_playground.clickview_task2
    ,lateral flatten( input => src:folders)
where id = '10';
```
## B)
```
-- flatten the JSON by libraries to access library ids and data
select 'libraries' as key, value:id::number as id, value:name::string as name from scar_playground.clickview_task2
    ,lateral flatten( input => src:libraries)
where id = '12'
union
-- Can flatten specifically to src since series only has 1 record and contains a key (would need to edit if series data expands)
select key, value:id::number as id, value:name::string as name from scar_playground.clickview_task2
    ,lateral flatten( input => src)
where key = 'series' and id = '14';
```

## Task 3
```-- Use security team or account admin account so appropriate owner appears in logs
USE ROLE ACCOUNTADMIN;

-- Create role with comment to it's use
create role if not exists BI_TOOL_ROLE comment = 'A read/write role for monitoring BI tool usage';

grant usage on warehouse BI_TOOL_WAREHOUSE to role BI_TOOL_ROLE;
grant usage on database CLICKVIEW_DB to role BI_TOOL_ROLE;

-- BI tool will be able to see and query all tables and views in the CLICKVIEW schema
grant usage on schema CLICKVIEW_DB.CLICKVIEW to role BI_TOOL_ROLE;
grant select on all tables in schema CLICKVIEW_DB.CLICKVIEW to role BI_TOOL_ROLE;
grant select on all views in schema CLICKVIEW_DB.CLICKVIEW to role BI_TOOL_ROLE;
grant select on future tables in schema CLICKVIEW_DB.CLICKVIEW to role BI_TOOL_ROLE;
grant select on future views in schema CLICKVIEW_DB.CLICKVIEW to role BI_TOOL_ROLE;

-- This role will be able to view queries and usage in the new warehouse as well as edit settings i.e. warehouse size
grant modify on warehouse BI_TOOL_WAREHOUSE to role BI_TOOL_ROLE;
grant monitor on warehouse BI_TOOL_WAREHOUSE to role BI_TOOL_ROLE;

-- Don't know if this role should be able to edit the tables and views, would require approval first
-- grant create table on schema CLICKVIEW_DB.CLICKVIEW to role BI_TOOL_ROLE;
-- grant create view on schema CLICKVIEW_DB.CLICKVIEW to role BI_TOOL_ROLE;
-- grant insert, update on all tables in schema CLICKVIEW_DB.CLICKVIEW to role BI_TOOL_ROLE;
-- grant insert, update on all views in schema CLICKVIEW_DB.CLICKVIEW to role BI_TOOL_ROLE;
-- grant insert, update on future tables in schema CLICKVIEW_DB.CLICKVIEW to role BI_TOOL_ROLE;
-- grant insert, update on future views in schema CLICKVIEW_DB.CLICKVIEW to role BI_TOOL_ROLE;

-- Assign the correct user to the new role
grant role BI_TOOL_ROLE to user "SEAN_CAR";

-- Allow the Looker role to see future views (not tables)
grant usage on schema CLICKVIEW_DB.CLICKVIEW to role LOOKER_ROLE;
grant select on future views in schema CLICKVIEW_DB.CLICKVIEW to role LOOKER_ROLE;
```
## Task 4

### Analysis of the query
The cluster key is the timestamp column truncated to date which makes for a great key as this means it has enough cardinality that snowflake can do effective pruning on the table but not so many that the same values can’t be grouped into the same micro-partitions. Around half of the micro-partitions are able constant and these aid in query performance as Snowflake can easily prune them when scanning the dataset. Overlaps are micro-partitions that contain the same values as other micro-partitions, making it harder for Snowflake to prune through them as it scans over the same values multiple times. Overlap depth is the amount of overlapping micro-partitions on a single value. This is the one that can greatly affect performance and should be kept to a minimum. Shown in the histogram, almost half the partitions have a depth of 1 which is great, the numbers further down are the concerning ones.

Credit: snowflake documentation - (https://docs.snowflake.com/en/_images/tables-clustering-ratio.png)

### How to improve performance

Re-clustering is done by Snowflake automatically (unlike what I said on my last attempt) but this process takes credits, especially on larger tables so there are some things to minimise how often this has to happen and improve average query performance:

-	Potentially add another cluster key with the same properties as I discussed above. This could be ‘TYPE’ depending on how many types there are (I don’t fully understand the definition of this field)
-	If this table contains ALL the ClickView data, then splitting the JSON data into separate flattened tables based on object (e.g. events, accounts, users, libraries, etc.) could be a cheaper way of querying for analytics and troubleshooting
-	Table clustering degrades the more DML performed on the table, lowering the update/merge rate on this table would help decrease the number of times you would need to re-cluster
-	One the same vain as lowering DML, splitting out this table based on requirements of this data’s lead time such as creating a table for real-time requirements and daily load requirements
- Creating views on top of this table which only looks at the most recent data is also a great tactic of increasing query performance I like to use

# Editor's Note
After my performance on the last tech test it was apparent that Snowflake's clustering and query optimisation was a definite gap in my knowledge so before attempting this second chance I did spend about an hour researching how they work. In the interest of setting accurate expectations of why technical skill, this task took me around 2 hours to complete excluding research done beforehand. I wanted to make sure I completely redid the reappearing questions, double checked everything and didn't rush.

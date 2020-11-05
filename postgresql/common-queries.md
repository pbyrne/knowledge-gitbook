# Common queries

## Perform an update from a select

## Finding the size of your biggest relations

Relations are objects in the database such as tables and indexes, and this query shows the size of all the individual parts. Tables which have both regular and TOAST pieces will be broken out into separate components; an example showing how you might include those into the main total is available in the documentation, and as of PostgreSQL 9.0 it's possible to include it automatically by using pg\_table\_size here instead of pg\_relation\_size:

Note that all of the queries below this point on this page show you the sizes for only those objects which are in the database you are currently connected to.

```sql
SELECT nspname || '.' || relname AS "relation",
  pg_size_pretty(pg_relation_size(C.oid)) AS "size"
FROM pg_class C
LEFT JOIN pg_namespace N ON (N.oid = C.relnamespace)
WHERE nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_relation_size(C.oid) DESC
LIMIT 20;
```

## Finding vacuum timestamps and live/dead tuples for tables

```sql
select psut.relname,
  to_char(psut.last_vacuum, 'yyyy-mm-dd hh24:mi') as last_vacuum,
  to_char(psut.last_autovacuum, 'yyyy-mm-dd hh24:mi') as last_autovacuum,
  to_char(pg_class.reltuples, '9g999g999g999') as n_tup,
  to_char(psut.n_dead_tup, '9g999g999g999') as dead_tup,
  to_char(cast(current_setting('autovacuum_vacuum_threshold') as bigint)
    + (cast(current_setting('autovacuum_vacuum_scale_factor') as numeric)
    * pg_class.reltuples), '9g999g999g999') as av_threshold,
  case
  when cast(current_setting('autovacuum_vacuum_threshold') as bigint)
    + (cast(current_setting('autovacuum_vacuum_scale_factor') as numeric)
    * pg_class.reltuples) < psut.n_dead_tup
  then '*'
  else ''
  end as expect_av
from pg_stat_user_tables psut
join pg_class on psut.relid = pg_class.oid
order by 1;
```

## Finding the total size of a given table

```sql
select relname, pg_size_pretty(pg_indexes_size(oid)) as index_size, pg_size_pretty(pg_relation_size(oid)) as data_size
from pg_class
where relname = 'screenshot_views';
```

```text
     relname      | index_size | data_size
------------------+------------+-----------
 screenshot_views | 6919 MB    | 9264 MB
```




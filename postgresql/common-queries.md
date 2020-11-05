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

## Largest tables \(data + index\)

```sql
select relname, pg_size_pretty(pg_indexes_size(pg_class.oid)) as index_size,
  pg_size_pretty(pg_relation_size(pg_class.oid)) as data_size,
  pg_size_pretty(pg_indexes_size(pg_class.oid) + pg_relation_size(pg_class.oid)) as total_size 
from pg_class 
join pg_namespace on pg_namespace.oid = pg_class.relnamespace 
where relkind = 'r'
  and pg_namespace.nspname not in ('pg_catalog', 'information_schema') 
order by pg_indexes_size(pg_class.oid) + pg_relation_size(pg_class.oid) desc 
limit 20; 
```

```text
        relname        | index_size | data_size | total_size 
-----------------------+------------+-----------+------------
 screenshots           | 39 GB      | 6413 MB   | 45 GB
 likes                 | 18 GB      | 6920 MB   | 25 GB
 users                 | 6952 MB    | 1843 MB   | 8796 MB
 user_sessions         | 3432 MB    | 4533 MB   | 7964 MB
 timeline_events       | 6889 MB    | 786 MB    | 7676 MB
 profile_views         | 3773 MB    | 2649 MB   | 6423 MB
 followings            | 4249 MB    | 2043 MB   | 6293 MB
 changes               | 1539 MB    | 4739 MB   | 6278 MB
 colors                | 3414 MB    | 2230 MB   | 5644 MB
 taggings              | 2797 MB    | 1465 MB   | 4261 MB
 notifications         | 1783 MB    | 1536 MB   | 3319 MB
 users_deleted         | 236 MB     | 2517 MB   | 2753 MB
 job_clicks            | 437 MB     | 1798 MB   | 2235 MB
 bucketings            | 1465 MB    | 676 MB    | 2141 MB
 comments_with_deleted | 1158 MB    | 780 MB    | 1938 MB
 stats_user_dailies    | 620 MB     | 687 MB    | 1307 MB
 webhooks              | 21 MB      | 1155 MB   | 1175 MB
 authentications       | 209 MB     | 941 MB    | 1150 MB
 buckets               | 911 MB     | 125 MB    | 1035 MB
 email_changes         | 439 MB     | 390 MB    | 829 MB
(20 rows)
```

## Displaying the current value of a setting

```sql
select current_setting('max_standby_streaming_delay');
```

```text
 current_setting 
-----------------
 10min
(1 row)
```

## Display currently running queries against a given table

```sql
select pid, query, state, locktype, mode, backend_start
from pg_locks
join pg_stat_activity
  using (pid)
where relation::regclass = 'email_events'::regclass
  and granted is true
  and backend_xmin is not null;
```

## Display long-running queries

```sql
SELECT
	pid,
	now() - pg_stat_activity.query_start AS duration,
	query AS query
FROM
	pg_stat_activity
WHERE
	pg_stat_activity.query <> ''::text
	AND state <> 'idle'
	AND now() - pg_stat_activity.query_start > interval '5 minutes'
ORDER BY now() - pg_stat_activity.query_start DESC
;
```

## Kill a query

Use one of the above to find the PID, and then:

```sql
SELECT pg_cancel_backend(THE_PID)
```




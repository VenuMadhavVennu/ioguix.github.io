---
layout: post
author: ioguix
title: Playing with indexes and better bloat estimate
date: 2014-03-28 07:05:00 +0100
tags: [bloat, postgresql]
category: postgresql
---
Most of the PostgreSQL DBAs might know about [this large bloat estimate query][http://wiki.postgresql.org/wiki/Show_database_bloat]
integrated in [check_postgres](http://bucardo.org/wiki/Check_postgres).  It is
supposed to compute a rough estimate of the bloat for tables and indexes in a
database.  As the PostgreSQL wiki page says:

> This query is for informational purposes only.  It provides a loose estimate
> of table growth activity only, and should not be construed as a 100% accurate
> portrayal of space consumed by database objects

Lately, a customer asked me about existing tools to help deciding when he is
supposed to `REINDEX`.  While writing a detailed answer with examples, I had
quite a surprise seeing an estimated index size ratio of 0.7 for a simple
table!  I realized I never payed much attention to this part of the result...

## The problem

Here is a test table, a copy table <code>rental</code> from
[pagila](http://pgfoundry.org/frs/download.php/1719/pagila-0.10.1.zip) project:

{% highlight psql %}
pagila=# CREATE TABLE test (LIKE rental INCLUDING DEFAULTS INCLUDING CONSTRAINTS INCLUDING INDEXES );
pagila=# INSERT INTO test SELECT * FROM rental;
pagila=# ANALYZE;
pagila=# \d test
                                           Table "public.test"
    Column    |            Type             |                         Modifiers                          
--------------+-----------------------------+------------------------------------------------------------
 rental_id    | integer                     | not null default nextval('rental_rental_id_seq'::regclass)
 rental_date  | timestamp without time zone | not null
 inventory_id | integer                     | not null
 customer_id  | smallint                    | not null
 return_date  | timestamp without time zone | 
 staff_id     | smallint                    | not null
 last_update  | timestamp without time zone | not null default now()
Indexes:
    "test_pkey" PRIMARY KEY, btree (rental_id)
    "test_rental_date_inventory_id_customer_id_idx" UNIQUE, btree (rental_date, inventory_id, customer_id)
    "test_inventory_id_idx" btree (inventory_id)
{% endhighlight %}

I copy pasted the bloat query in file <code>~/tmp/bloat_original.sql</code>, here is its output for this table:
{% highlight psql %}
postgres@pagila=# \i ~/tmp/bloat_original.sql
 current_database | schemaname |    tablename     | tbloat | wastedbytes |                        iname                        | ibloat | wastedibytes 
------------------+------------+------------------+--------+-------------+-----------------------------------------------------+--------+--------------
...
 pagila           | public     | test             |    1.2 |      188416 | test_pkey                                           |    0.5 |            0
 pagila           | public     | test             |    1.2 |      188416 | test_rental_date_inventory_id_customer_id_idx       |    0.8 |            0
 pagila           | public     | test             |    1.2 |      188416 | test_inventory_id_idx                               |    0.7 |            0
...
{% endhighlight %}

A B-tree index is filled at 90% per default, see storage parameter
[FILL FACTOR](http://www.postgresql.org/docs/9.3/static/sql-createindex.html#SQL-CREATEINDEX-STORAGE-PARAMETERS).
So, a freshly created B-tree index with no bloat is supposed to be 1.1x larger
than it should be with a `FILL FACTOR` of 100.  As I had some time
to dive in this, I couldn't resist to investigate these insane estimated size
factors for my test table indexes: 0.5, 0.8 and 0.7.  How an index could be
smaller than it is supposed to be ?

I hadn't to dive deap in the query, after a closer look at the query code and
comments, we find:

{% highlight sql %}
COALESCE(CEIL((c2.reltuples*(datahdr-12))/(bs-20::float)),0) AS iotta -- very rough approximation, assumes all cols
{% endhighlight %}

Oh, ok. The query estimates the ideal size of each index considering it
references *ALL* the table fields.  It's quite rare to find a btree index on
all fields of a table, and obviously there's no point having multiple indexes
on a table, all of them referencing all fields of the table.

## A look at the real bloat

First, let see how the indexes are really bloated:

{% highlight psql %}
pagila=# create schema stattuple;
pagila=# create extension pgstattuple with schema stattuple;
pagila=# SELECT relname, pg_table_size(oid) as index_size,
  100-(stattuple.pgstatindex(relname)).avg_leaf_density AS bloat_ratio
FROM pg_class
WHERE relname ~ 'test' AND relkind = 'i';

                    relname                    | index_size | bloat_ratio 
-----------------------------------------------+------------+-------------
 test_pkey                                     |     376832 |       10.25
 test_inventory_id_idx                         |     507904 |       34.11
 test_rental_date_inventory_id_customer_id_idx |     630784 |       26.14
{% endhighlight %}

First point, the bloat on indexes is not 10% everywhere, only for the PK. This
is because indexes were created *BEFORE* inserting data.  So it looks like data
were naturally order on the PK on table "rental" when scanning it sequentially.
What if we load data sorting on "inventory_id" field ?

{% highlight psql %}
pagila=# TRUNCATE test;
pagila=# INSERT INTO test SELECT * FROM rental ORDER BY inventory_id ;
pagila=# SELECT relname, pg_table_size(oid) as index_size,
  100-(stattuple.pgstatindex(relname)).avg_leaf_density AS bloat_ratio
FROM pg_class
WHERE relname ~ 'test' AND relkind = 'i';

                    relname                    | index_size | bloat_ratio 
-----------------------------------------------+------------+-------------
 test_pkey                                     |     524288 |       36.22
 test_inventory_id_idx                         |     376832 |       10.25
 test_rental_date_inventory_id_customer_id_idx |     647168 |       28.04
{% endhighlight %}

Ok, it totally makes sense there: index `test_inventory_id_idx` is now 10%
bloated.  First lesson: Do not create your indexes before loading your datas!
Demo:

{% highlight psql %}
pagila=# DROP TABLE test;
pagila=# CREATE TABLE test (LIKE rental INCLUDING DEFAULTS INCLUDING CONSTRAINTS);
pagila=# INSERT INTO test SELECT * FROM rental;
pagila=# ALTER TABLE test ADD CONSTRAINT test_pkey PRIMARY KEY (rental_id); 
pagila=# CREATE INDEX test_inventory_id_idx ON test (inventory_id);
pagila=# CREATE INDEX test_rental_date_inventory_id_customer_id_idx ON test (rental_date, inventory_id, customer_id);

pagila=# SELECT relname, pg_table_size(oid) as index_size,
  100-(stattuple.pgstatindex(relname)).avg_leaf_density AS bloat_ratio
FROM pg_class
WHERE relname ~ 'test' AND relkind = 'i';

                    relname                    | index_size | bloat_ratio 
-----------------------------------------------+------------+-------------
 test_pkey                                     |     376832 |       10.25
 test_inventory_id_idx                         |     376832 |       10.25
 test_rental_date_inventory_id_customer_id_idx |     524288 |       10.73
{% endhighlight %}

## A better query to estimate index bloat?

> Wait, you just showed us you can have the real bloat on indexes, why would I
> want a loose estimate from a query relying on stats?!

Because I feel bad reading the whole index again and again from monitoring
tools every few minutes.  Having a loose estimate is good enough in some cases.
And anyway, I want to dig into this for education :-)

So, I tried to write a query able to give a better estimate of bloat *ONLY* for
indexes.  I picked some parts of the original bloat query, but rewrote mostly
everything.  Here is the result:

{% highlight sql %}
-- change to the max number of field per index if not default.
\set index_max_keys 32
-- (readonly) IndexTupleData size
\set index_tuple_hdr 2
-- (readonly) ItemIdData size
\set item_pointer 4
-- (readonly) IndexAttributeBitMapData size
\set index_attribute_bm (:index_max_keys + 8 - 1) / 8

SELECT current_database(), nspname, c.relname AS table_name, index_name, bs*(sub.relpages)::bigint AS totalbytes,
  CASE WHEN sub.relpages <= otta THEN 0 ELSE bs*(sub.relpages-otta)::bigint END                                    AS wastedbytes,
  CASE WHEN sub.relpages <= otta THEN 0 ELSE bs*(sub.relpages-otta)::bigint * 100 / (bs*(sub.relpages)::bigint) END AS realbloat
FROM (
  SELECT bs, nspname, table_oid, index_name, relpages, coalesce(
    ceil((reltuples*(:item_pointer+nulldatahdrwidth))/(bs-pagehdr::float)) +
      CASE WHEN am.amname IN ('hash','btree') THEN 1 ELSE 0 END , 0 -- btree and hash have a metadata reserved block
    ) AS otta
  FROM (
    SELECT maxalign, bs, nspname, relname AS index_name, reltuples, relpages, relam, table_oid,
      ( index_tuple_hdr_bm +
          maxalign - CASE /* Add padding to the index tuple header to align on MAXALIGN */
            WHEN index_tuple_hdr_bm%maxalign = 0 THEN maxalign
            ELSE index_tuple_hdr_bm%maxalign
          END
        + nulldatawidth + maxalign - CASE /* Add padding to the data to align on MAXALIGN */
            WHEN nulldatawidth::integer%maxalign = 0 THEN maxalign
            ELSE nulldatawidth::integer%maxalign
          END
      )::numeric AS nulldatahdrwidth, pagehdr
    FROM (
      SELECT
        i.nspname, i.relname, i.reltuples, i.relpages, i.relam, s.starelid, a.attrelid AS table_oid,
        current_setting('block_size')::numeric AS bs,
        /* MAXALIGN: 4 on 32bits, 8 on 64bits (and mingw32 ?) */
        CASE
          WHEN version() ~ 'mingw32' OR version() ~ '64-bit' THEN 8
          ELSE 4
        END AS maxalign,
        /* per page header, fixed size: 20 for 7.X, 24 for others */
        CASE WHEN substring(current_setting('server_version') FROM '#"[0-9]+#"%' FOR '#')::integer > 7
          THEN 24
          ELSE 20
        END AS pagehdr,
        /* per tuple header: add index_attribute_bm if some cols are null-able */
        CASE WHEN max(coalesce(s.stanullfrac,0)) = 0
          THEN :index_tuple_hdr
          ELSE :index_tuple_hdr + :index_attribute_bm
        END AS index_tuple_hdr_bm,
        /* data len: we remove null values save space using it fractionnal part from stats */
        sum( (1-coalesce(s.stanullfrac, 0)) * coalesce(s.stawidth, 2048) ) AS nulldatawidth
      FROM pg_attribute AS a
        JOIN pg_statistic AS s ON s.starelid=a.attrelid AND s.staattnum = a.attnum
        JOIN (
          SELECT nspname, relname, reltuples, relpages, indrelid, relam, regexp_split_to_table(indkey::text, ' ')::smallint AS attnum
          FROM pg_index
            JOIN pg_class ON pg_class.oid=pg_index.indexrelid
            JOIN pg_namespace ON pg_namespace.oid = pg_class.relnamespace
        ) AS i ON i.indrelid = a.attrelid AND a.attnum = i.attnum
      WHERE a.attnum > 0
      GROUP BY 1, 2, 3, 4, 5, 6, 7, 8, 9
    ) AS s1
  ) AS s2
    LEFT JOIN pg_am am ON s2.relam = am.oid
) as sub
JOIN pg_class c ON c.oid=sub.table_oid
ORDER BY 2,3,4
{% endhighlight %}

How does it perform compared to the actual values? Let's go back to a bloated
situation: 

{% highlight psql %}
pagila=# TRUNCATE test ;
pagila=# INSERT INTO test SELECT * FROM rental;
pagila=# SELECT relname, pg_table_size(oid) as index_size,
  100-(stattuple.pgstatindex(relname)).avg_leaf_density AS bloat_ratio
FROM pg_class
WHERE relname ~ 'test' AND relkind = 'i'
ORDER BY 1;
pagila=# ANALYZE test;

                    relname                    | index_size | bloat_ratio 
-----------------------------------------------+------------+-------------
 test_inventory_id_idx                         |     507904 |       34.11
 test_pkey                                     |     376832 |       10.25
 test_rental_date_inventory_id_customer_id_idx |     630784 |       26.14
{% endhighlight %}

I created the file "~/tmp/bloat_index.sql" with this estimated indexes bloat
query filtering on table `test`, here is its result:

{% highlight psql %}
pagila=# \i ~/tmp/bloat_index.sql 
 current_database | nspname | table_name |                  index_name                   | totalbytes | wastedbytes |      realbloat      
------------------+---------+------------+-----------------------------------------------+------------+-------------+---------------------
 pagila           | public  | test       | test_inventory_id_idx                         |     507904 |      172032 | 33.8709677419354839
 pagila           | public  | test       | test_pkey                                     |     376832 |       40960 | 10.8695652173913043
 pagila           | public  | test       | test_rental_date_inventory_id_customer_id_idx |     630784 |      172032 | 27.2727272727272727
{% endhighlight %}

Well, pretty close :-)

Let's REINDEX:

{% highlight psql %}
pagila=# REINDEX TABLE test;
pagila=# ANALYZE test;
pagila=# SELECT relname, pg_table_size(oid) as index_size,
  100-(stattuple.pgstatindex(relname)).avg_leaf_density AS bloat_ratio
FROM pg_class
WHERE relname ~ 'test' AND relkind = 'i'
ORDER BY 1;

                    relname                    | index_size | bloat_ratio 
-----------------------------------------------+------------+-------------
 test_inventory_id_idx                         |     376832 |       10.25
 test_pkey                                     |     376832 |       10.25
 test_rental_date_inventory_id_customer_id_idx |     524288 |       10.73

pagila=# \i ~/tmp/bloat_index.sql 
 current_database | nspname | table_name |                  index_name                   | totalbytes | wastedbytes |      realbloat      
------------------+---------+------------+-----------------------------------------------+------------+-------------+---------------------
 pagila           | public  | test       | test_inventory_id_idx                         |     376832 |       40960 | 10.8695652173913043
 pagila           | public  | test       | test_pkey                                     |     376832 |       40960 | 10.8695652173913043
 pagila           | public  | test       | test_rental_date_inventory_id_customer_id_idx |     524288 |       65536 | 12.5000000000000000
{% endhighlight %}

Not bad. 

Let's create some bloat. The table is approx. 160,000 rows, so the following
query updates ~10% of the table:

{% highlight psql %}
pagila=# UPDATE test SET rental_date = current_timestamp WHERE rental_id < 3200 AND rental_id%2 = 0;
pagila=# ANALYZE test ;

pagila=# SELECT relname, pg_table_size(oid) as index_size,
  100-(stattuple.pgstatindex(relname)).avg_leaf_density AS bloat_ratio
FROM pg_class
WHERE relname ~ 'test' AND relkind = 'i'
ORDER BY 1;

                    relname                    | index_size | bloat_ratio 
-----------------------------------------------+------------+-------------
 test_inventory_id_idx                         |     475136 |       22.42
 test_pkey                                     |     450560 |       18.04
 test_rental_date_inventory_id_customer_id_idx |     598016 |       14.26
(3 rows)

pagila=# \i ~/tmp/bloat_index.sql 
 current_database | nspname | table_name |                  index_name                   | totalbytes | wastedbytes |      realbloat      
------------------+---------+------------+-----------------------------------------------+------------+-------------+---------------------
 pagila           | public  | test       | test_inventory_id_idx                         |     475136 |      139264 | 29.3103448275862069
 pagila           | public  | test       | test_pkey                                     |     450560 |      114688 | 25.4545454545454545
 pagila           | public  | test       | test_rental_date_inventory_id_customer_id_idx |     598016 |      139264 | 23.2876712328767123
(3 rows)
{% endhighlight %}

Erk, not so good. Moar bloat?

{% highlight psql %}
pagila=# UPDATE test SET rental_date = current_timestamp WHERE rental_id%2 = 0;
pagila=# ANALYZE test ;
pagila=# SELECT relname, pg_table_size(oid) as index_size,
  100-(stattuple.pgstatindex(relname)).avg_leaf_density AS bloat_ratio
FROM pg_class
WHERE relname ~ 'test' AND relkind = 'i'
ORDER BY 1;

                    relname                    | index_size | bloat_ratio 
-----------------------------------------------+------------+-------------
 test_inventory_id_idx                         |     745472 |       55.48
 test_pkey                                     |     737280 |       54.98
 test_rental_date_inventory_id_customer_id_idx |     925696 |       49.96
(3 rows)

pagila=# \i ~/tmp/bloat_index.sql 
 current_database | nspname | table_name |                  index_name                   | totalbytes | wastedbytes |      realbloat      
------------------+---------+------------+-----------------------------------------------+------------+-------------+---------------------
 pagila           | public  | test       | test_inventory_id_idx                         |     745472 |      409600 | 54.9450549450549451
 pagila           | public  | test       | test_pkey                                     |     737280 |      401408 | 54.4444444444444444
 pagila           | public  | test       | test_rental_date_inventory_id_customer_id_idx |     925696 |      466944 | 50.4424778761061947
(3 rows)
{% endhighlight %}

Well, better again.

## Conclusion

This new query performs much better for indexes than the usual one everyone
knew for a long time.  But it isn't perfect, showing sometimes some quite wrong
results.

First issue, this query doesn't make any difference between branch nodes and
leaves in indexes.  Rows values and references are kept in leaves, and only a
fractional part of them are in branches.  I should do some more tests with
larger indexes to see how it behaves with a lot of branch nodes.  As the number
of branch nodes is supposed to be a logarithm of the total number of rows,
maybe we could include some more guessing in the optimal index size computing.

Second issue...keep in mind it is based on statistics.

Anyway, as useful as this query can be, at least it has been funny to dive in
indexes and pages guts.

As a side note, I had been surprised to see that padding to `MAXALIGN` was
applied on both tuple header AND tuple data.  My understanding is that for each
row in an index leave page, we have:
* the item_pointer at the beginning of the page,
* the tuple header + data at the end of the page that must be aligned on
  `MAXALIGN`.

Why ain't they padded *together* to `MAXALIGN`? Wouldn't this save
some space? 

Thank you for reading till the end! Cheers, see you maybe in another post about
table bloat next time ;-) 

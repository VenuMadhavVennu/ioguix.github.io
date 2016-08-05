---
layout: post
author: ioguix
title: Bloat estimation for tables
date: 2014-09-10 18:30:00 +0200
tags: [bloat, postgresql]
category: postgresql
---

After my [Btree bloat estimation query]({%post_url 2014-09-10-Bloat-estimation-for-tables%}),
I found some time to work on a new query for tables. The goal here is still to
have a better bloat estimation using dedicated queries for each kind of objects.

Compare to "the well known":http://wiki.postgresql.org/wiki/Show_database_bloat
bloat query, this query pay attention to:

* TOAST
* headers of variable length types
* easier to filter or parse

You'll find the queries here:

* from PostgreSQL 7.4 to 8.1: [https://gist.github.com/ioguix/f849b1bd31be55da2d7f]
* from PostgreSQL 8.2 to 8.4: [https://gist.github.com/ioguix/74769c8fe5edc582a61b]
* for PostgreSQL 9.0 and after: [https://gist.github.com/ioguix/4f95917f90c9e26df1b2]

##Tests

I created the file @sql/bloat_tables.sql@ with the 9.0 and more query version.
I edited the query to add the bloat reported by `pgstattuple` (free_percent +
dead_tuple_percent) to compare both results and added the following filter:

```sql
-- remove Non Applicable tables
NOT is_na
-- remove tables with real bloat < 1 block
AND tblpages*((pst).free_percent + (pst).dead_tuple_percent)::float4/100 >= 1
-- filter on table name using the parameter :tblname
AND tblname LIKE :'tblname'
```

Here is the result on a fresh pagila database:

```
postgres@pagila=# \set tblname %
postgres@pagila=# \i sql/bloat_tables.sql 
 current_database | schemaname |    tblname     | real_size | bloat_size | tblpages | is_na |   bloat_ratio    | real_frag 
------------------+------------+----------------+-----------+------------+----------+-------+------------------+-----------
 pagila           | pg_catalog | pg_description |    253952 |       8192 |       31 | f     |  3.2258064516129 |      3.34
 pagila           | public     | city           |     40960 |       8192 |        5 | f     |               20 |     20.01
 pagila           | public     | customer       |     73728 |       8192 |        9 | f     | 11.1111111111111 |     11.47
 pagila           | public     | film           |    450560 |       8192 |       55 | f     | 1.81818181818182 |      3.26
 pagila           | public     | rental         |   1228800 |     131072 |      150 | f     | 10.6666666666667 |      0.67
(5 rows)
```

Well, not too bad. Let's consider the largest table, clone it and create some
bloat:

```
postgres@pagila=# create table film2 as select * from film;
SELECT 1000
postgres@pagila=# analyze film2;
ANALYZE
postgres@pagila=# \set tblname film%
postgres@pagila=# \i sql/bloat_tables.sql
 current_database | schemaname | tblname | real_size | bloat_size | tblpages | is_na |   bloat_ratio    | real_frag 
------------------+------------+---------+-----------+------------+----------+-------+------------------+-----------
 pagila           | public     | film    |    450560 |       8192 |       55 | f     | 1.81818181818182 |      3.26
 pagila           | public     | film2   |    450560 |       8192 |       55 | f     | 1.81818181818182 |      3.26
(2 rows)


postgres@pagila=# delete from film2 where film_id < 250;
DELETE 249
postgres@pagila=# analyze film2;
ANALYZE
postgres@pagila=# \set tblname film2
postgres@pagila=# \i sql/bloat_tables.sql
 current_database | schemaname | tblname | real_size | bloat_size | tblpages | is_na |   bloat_ratio    | real_frag 
------------------+------------+---------+-----------+------------+----------+-------+------------------+-----------
 pagila           | public     | film2   |    450560 |     122880 |       55 | f     | 27.2727272727273 |     27.29
(1 row)
```

Again, the bloat reported here is pretty close to the reality!

Some more tests:

```
postgres@pagila=# delete from film2 where film_id < 333;
DELETE 83
postgres@pagila=# analyze film2;
ANALYZE
postgres@pagila=# \i sql/bloat_tables.sql
 current_database | schemaname | tblname | real_size | bloat_size | tblpages | is_na |   bloat_ratio    | real_frag 
------------------+------------+---------+-----------+------------+----------+-------+------------------+-----------
 pagila           | public     | film2   |    450560 |     155648 |       55 | f     | 34.5454545454545 |     35.08
(1 row)

postgres@pagila=# delete from film2 where film_id < 666;
DELETE 333
postgres@pagila=# analyze film2;
ANALYZE
postgres@pagila=# \i sql/bloat_tables.sql
 current_database | schemaname | tblname | real_size | bloat_size | tblpages | is_na |   bloat_ratio    | real_frag 
------------------+------------+---------+-----------+------------+----------+-------+------------------+-----------
 pagila           | public     | film2   |    450560 |     303104 |       55 | f     | 67.2727272727273 |     66.43
(1 row)
```

Good, good, good. What next?


##The alignment deviation

You might have noticed I did not mentioned this table with a large deviation
between the statistical bloat and the real one, called "rental":

```
postgres@pagila=# \set tblname rental
postgres@pagila=# \i sql/bloat_tables.sql
 current_database | schemaname | tblname | real_size | bloat_size | tblpages | is_na |   bloat_ratio    | real_frag 
------------------+------------+---------+-----------+------------+----------+-------+------------------+-----------
 pagila           | public     | rental  |   1228800 |     131072 |      150 | f     | 10.6666666666667 |      0.67
(1 row)
```

This particular situation is exactly why I loved writing these bloat queries
(including the btree one), confronting the statistics and the reality and
finding a logical answer or a fix.

Statistical and real bloat are actually both right here. The statistical one is
just measuring here the bloat AND something else we usually don't pay attention
to. I'll call it the alignment overhead.

Depending on the fields types, PostgreSQL adds some padding before the values
to align them inside the row in regards to the CPU word size. This help
ensuring a value fits in only one CPU register when possible. Alignment padding
are given in [this pg_type](http://www.postgresql.org/docs/current/static/catalog-pg-type.html)
page from PostgreSQL document, see field `typalign`.

So let's demonstrate how it influence the bloat here. Back to the rental table,
here is its definition: 

```
postgres@pagila=# \d rental
                                          Table "public.rental"
    Column    |            Type             |                         Modifiers                          
--------------+-----------------------------+------------------------------------------------------------
 rental_id    | integer                     | not null default nextval('rental_rental_id_seq'::regclass)
 rental_date  | timestamp without time zone | not null
 inventory_id | integer                     | not null
 customer_id  | smallint                    | not null
 return_date  | timestamp without time zone | 
 staff_id     | smallint                    | not null
 last_update  | timestamp without time zone | not null default now()
```

All the fields here are fixed-size types, so it is quite easy to compute the
row size:

* `rental_id` and `inventory_id` are 4-bytes integers, possible alignment is
  every 4 bytes from the begining of the row
* `customer_id` and `staff_id` are 2-bytes integers, possible alignment is
  every 2 bytes from the begining of the row
* `rental_date`, `return_date` and `last_update` are 8-bytes timestamps,
  possible alignment is every 8 bytes from the begining of the row

The minimum row size would be `2*4 + 2*2 + 3*8`, 36 bytes. Considering the
alignment optimization and the order of the fields, we now have (ascii art is
easier to explain):

```
|0     1     2     3     4     5     6     7     8     |
|       rental_id       |***********PADDING************|
|                     rental_date                      |
|     inventory_id      |customer_id|******PADDING*****|
|                     return_date                      |
| staff_id  |*****************PADDING******************|
|                     last_update                      |
```

That makes 12 bytes of padding and a total row size of 48 bytes instead of 36.
Here are the 10%! Let's double check this by the experience: 

```
postgres@pagila=# create table rental2 as select rental_date, return_date,
last_update, rental_id, inventory_id, customer_id, staff_id from public.rental; 
SELECT 16044
postgres@pagila=# \d rental2
                 Table "public.rental2"
    Column    |            Type             | Modifiers 
--------------+-----------------------------+-----------
 rental_date  | timestamp without time zone | 
 return_date  | timestamp without time zone | 
 last_update  | timestamp without time zone | 
 rental_id    | integer                     | 
 inventory_id | integer                     | 
 customer_id  | smallint                    | 
 staff_id     | smallint                    | 

postgres@pagila=# \dt+ rental*
                      List of relations
 Schema |  Name   | Type  |  Owner   |  Size   | Description 
--------+---------+-------+----------+---------+-------------
 public | rental  | table | postgres | 1200 kB | 
 public | rental2 | table | postgres | 1072 kB | 
(2 rows)


postgres@pagila=# select 100*(1200-1072)::float4/1200;
     ?column?     
------------------
 10.6666666666667
(1 row)
```

Removing the "remove tables with real bloat < 1 block" filter from my demo
query, we have now:

```
postgres@pagila=# \set tblname rental%
postgres@pagila=# \i sql/bloat_tables.sql
 current_database | schemaname | tblname | real_size | bloat_size | tblpages | is_na |   bloat_ratio    | real_frag 
------------------+------------+---------+-----------+------------+----------+-------+------------------+-----------
 pagila           | public     | rental  |   1228800 |     131072 |      150 | f     | 10.6666666666667 |      0.67
 pagila           | public     | rental2 |   1097728 |          0 |      134 | f     |                0 |      0.41
(2 rows)
```

Great!

Sadly, I couldn't find a good way to measure this in the queries so far, so I
will live with that. By the way, this alignment overhead might be a nice
subject for a script measuring it per tables.


##Known issues

The same than for the Btree statistical bloat query: I'm pretty sure the query
will have a pretty bad estimation with array types. I'll investigate about that
later.

Cheers, and happy monitoring!

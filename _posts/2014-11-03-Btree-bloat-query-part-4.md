---
layout: post
author: ioguix
title: Btree bloat query - part 4
date: 2014-11-03 13:39:00 +0200
tags: [bloat, postgresql]
category: postgresql
---

Thanks to the various PostgreSQL environments we have under monitoring at
Dalibo, these Btree bloat estimation queries keeps challenging me occasionally
because of statistics deviation...or bugs.

For people who visit this blog for the first time, don't miss the three
previous parts, stuffed with some interesting infos about these queries **and**
BTree indexes:
[part 1]({%post_url 2014-03-28-Playing-with-indexes-and-better-bloat-estimate%}),
[part 2]({%post_url 2014-06-24-More-work-on-index-bloat-estimation-query%}) and
[part 3]({%post_url 2014-09-09-Btree-bloat-query-changelog-part-3%}).

For people in a hurry, here are the links to the queries:

* for 7.4: [https://gist.github.com/ioguix/dfa41eb0ef73e1cbd943]()
* for 8.0 and 8.1: [https://gist.github.com/ioguix/5f60e24a77828078ff5f]()
* for 8.2 and more: [https://gist.github.com/ioguix/c29d5790b8b93bf81c27]()

##Columns has been ignored

In two different situations, some index fields were just ignored by the query:

* after renaming the field in the table
* if the index field was an expression

I cheated a bit for the first fix, looking at psql's answer to this question
(thank you `-E`).

The second one was an easy fix, but sadly only for version 8.0 and more. It
seems to me there's no solution for 7.4.

These bugs have the same results: very bad estimation. An index field is
ignored in both cases, s the bloat sounds much bigger with the old version of
the query. Here is a demo with an index on expression:

{% highlight psql %}
postgres@pagila=# create index test_expression on test (rental_id, md5(rental_id::text));
CREATE INDEX

postgres@pagila=# analyze ;
ANALYZE

postgres@pagila=# \i old/btree_bloat.sql-20141022
 current_database | schemaname | tblname |     idxname     | real_size | estimated_size | bloat_size |        bloat_ratio         | is_na 
------------------+------------+---------+-----------------+-----------+----------------+------------+----------------------------+-------
 pagila           | public     | test    | test_expression |    974848 |         335872 |     638976 |        65.5462184873949580 | f
{% endhighlight %}

Most of this 65% bloat estimation are actually the data of the missing field.
The result is much more coherent with the latest version of the query for a
freshly created index, supposed to have around 10% of bloat as showed in the
2nd query:

{% highlight psql %}
postgres@pagila=# \i sql/btree_bloat.sql
 current_database | schemaname | tblname |     idxname     | real_size | estimated_size | bloat_size |   bloat_ratio    | is_na 
------------------+------------+---------+-----------------+-----------+----------------+------------+------------------+-------
 pagila           | public     | test    | test_expression |    974848 |         851968 |     122880 | 12.6050420168067 | f

postgres@pagila=# SELECT relname, 100-(stattuple.pgstatindex(relname)).avg_leaf_density AS bloat_ratio
FROM pg_class WHERE relname = 'test_expression';
     relname     | bloat_ratio 
-----------------+-------------
 test_expression |       10.33
{% endhighlight %}

##Wrong estimation for varlena types

After fixing the query for indexes on expression, I noticed some negative bloat
estimation for the biggest ones: the real index was smaller than the estimated
one!

{% highlight psql %}
postgres@pagila=# create table test3 as select i from generate_series(1,10000000) i;
SELECT 10000000

postgres@pagila=# create index on test3(i, md5(i::text));
CREATE INDEX

postgres@pagila=# \i ~/sql/old/btree_bloat.sql-20141027
 current_database | schemaname | tblname |     idxname     | real_size | estimated_size | bloat_size |     bloat_ratio     | is_na 
------------------+------------+---------+-----------------+-----------+----------------+------------+---------------------+-------
 pagila           | public     | test3   | test3_i_md5_idx | 590536704 |      601776128 |  -11239424 | -1.9032557881448805 | f
{% endhighlight %}

In this version of the query, I am computing and adding the headers length of
varlena types (text, bytea, etc) to the statistics(see
[part 3]({%post_url 2014-09-09-Btree-bloat-query-changelog-part-3%})). I was
wrong.

Taking the "text" type as example, PostgreSQL adds a one byte header to the
value if it is not longer than 127, and a 4 bytes one for bigger ones. Looking
closer to the statistic values because of this negative bloat, I realized that
the headers was already added to them. As a demo, take a `md5` string of 32
bytes long. In the following results, we can see the average length from
`pg_stats` is `32+1` for one md5, and `4*32+4` for a string of 4 concatenated
md5, supposed to be 128 byte long:

{% highlight psql %}
postgres@pagila=# create table test2 as select i, md5(i::text), repeat(md5(i::text), 4) from generate_series(1,5) i;
SELECT 5

postgres@pagila=# analyze test2;
ANALYZE

postgres@pagila=# select tablename, attname, avg_width from pg_stats where tablename='test2';
 tablename | attname | avg_width 
-----------+---------+-----------
 test2     | i       |         4
 test2     | md5     |        33
 test2     | repeat  |       132
{% endhighlight %}

After removing this part of the query, stats for `test3_i_md5_idx` are much better:

{% highlight psql %}
postgres@pagila=# SELECT relname,
  100-(stattuple.pgstatindex(relname)).avg_leaf_density AS bloat_ratio
FROM pg_class WHERE relname = 'test3_i_md5_idx';
     relname     | bloat_ratio 
-----------------+-------------
 test3_i_md5_idx |       10.01

postgres@pagila=# \i ~/sql/old/btree_bloat.sql-20141028
 current_database | schemaname | tblname |     idxname     | real_size | estimated_size | bloat_size |     bloat_ratio     | is_na 
------------------+------------+---------+-----------------+-----------+----------------+------------+---------------------+-------
 pagila           | public     | test3   | test3_i_md5_idx | 590536704 |      521535488 |   69001216 | 11.6844923495221052 | f
{% endhighlight %}

This is a nice bug fix AND one complexity out of the query. Code simplification is always a good news :)

##Adding a bit of Opaque Data

When studying the Btree layout, I forgot about one small non-data area in index
pages: the "Special space", aka. "Opaque Data" in code sources. The previous
bug took me back on [this doc page](http://www.postgresql.org/docs/current/static/storage-page-layout.html)
where I remembered I should probably pay attention to this space.

This is is a small space on each pages reserved to the access method so it can
store whatever it needs for its own purpose. As instance, in the case of a
Btree index, this "special space" is 16 bytes long and used (among other
things) to reference both siblings of the page in the tree. Ordinary tables
have no opaque data, so no special space (good, I 'll not have to fix this bug
in my [Table bloat estimation query]({%post_url 2014-09-10-Bloat-estimation-for-tables %})).

This small bug is not as bad for stats than previous ones, but fixing it
definitely help the bloat estimation accuracy. Using the previous demo on
`test3_i_md5_idx`, here is the comparison of real bloat, estimation without
considering the special space and estimation considering it:

{% highlight psql %}
postgres@pagila=# SELECT relname,
  100-(stattuple.pgstatindex(relname)).avg_leaf_density AS bloat_ratio
FROM pg_class WHERE relname = 'test3_i_md5_idx';
     relname     | bloat_ratio 
-----------------+-------------
 test3_i_md5_idx |       10.01

postgres@pagila=# \i ~/sql/old/btree_bloat.sql-20141028
 current_database | schemaname | tblname |     idxname     | real_size | estimated_size | bloat_size |     bloat_ratio     | is_na 
------------------+------------+---------+-----------------+-----------+----------------+------------+---------------------+-------
 pagila           | public     | test3   | test3_i_md5_idx | 590536704 |      521535488 |   69001216 | 11.6844923495221052 | f

postgres@pagila=# \i ~/sql/btree_bloat.sql
 current_database | schemaname | tblname |     idxname     | real_size | estimated_size | bloat_size |   bloat_ratio    | is_na 
------------------+------------+---------+-----------------+-----------+----------------+------------+------------------+-------
 pagila           | public     | test3   | test3_i_md5_idx | 590536704 |      525139968 |   65396736 | 11.0741187731491 | f
{% endhighlight %}

This is only an approximative 5% difference for the estimated size of this particular index.

##Conclusion

I never mentioned it before, but these queries are used in
[check_pgactivity](https://github.com/OPMDG/check_pgactivity) (a nagios plugin
for PostgreSQL), under the checks "table_bloat" and "btree_bloat".
[The latest version](https://github.com/OPMDG/check_pgactivity/releases) of
this tool already include these fixes. I might write an article about
"check_pgactivity" at some point.

As it is not really convenient for most of you to follow the updates on my
gists, I keep writing here about my work on these queries. I should probably
add some version-ing on theses queries now and find a better way to communicate
about them at some point.

As a first step, after a discussion with (one of?) the author of
[pgObserver](http://zalando.github.io/PGObserver/) during the latest
[pgconf.eu](http://2014.pgconf.eu), I added these links to the following
PostgreSQL wiki pages:

* [https://wiki.postgresql.org/wiki/Index_Maintenance#New_query](https://wiki.postgresql.org/wiki/Index_Maintenance#New_query)
* [https://wiki.postgresql.org/wiki/Show_database_bloat](https://wiki.postgresql.org/wiki/Show_database_bloat)

Cheers, happy monitoring, happy REINDEX-ing!

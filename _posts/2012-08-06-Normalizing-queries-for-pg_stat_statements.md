---
layout: post
author: ioguix
title: Normalizing queries for pg_stat_statements < 9.2
date: 2012-08-06 19:53:00 +0200
tags: [postgresql]
category: postgresql
---
Hey,

If you follow PostgreSQL's development or [Depesz' blog](http://www.depesz.com/2012/03/30/waiting-for-9-2-pg_stat_statements-improvements/),
you might know that [pg_stat_statement](http://www.postgresql.org/docs/9.1/static/pgstatstatements.html)
extension is getting a lot of improvement in 9.2 and especially is able to
«lump "similar" queries together».  I will not re-phrase here what Despsz
already explain on his blog.

So, we have this great feature in 9.2, but what about previous release?  Until
9.1, pg_stat_statement is keeping track of most frequent queries individually.
No normalization, nothing.  It's been a while I've been thinking about
importing pgBadger normalization code in SQL.  Next pieces of code are tested
under PostgreSQL 9.1 but should be easy to port to previous versions.  So here
is the function to create (I tried my best to keep it readable :-)):

```sql
CREATE OR REPLACE FUNCTION normalize_query(IN TEXT, OUT TEXT) AS $body$
  SELECT
    regexp_replace(regexp_replace(regexp_replace(regexp_replace(
    regexp_replace(regexp_replace(regexp_replace(regexp_replace(

    lower($1),
    
    -- Remove extra space, new line and tab caracters by a single space
    '\s+',                          ' ',           'g'   ),

    -- Remove string content                       
    $$\\'$$,                        '',            'g'   ),
    $$'[^']*'$$,                    $$''$$,        'g'   ),
    $$''('')+$$,                    $$''$$,        'g'   ),

    -- Remove NULL parameters                      
    '=\s*NULL',                     '=0',          'g'   ),

    -- Remove numbers                              
    '([^a-z_$-])-?([0-9]+)',        '\1'||'0',     'g'   ),

    -- Remove hexadecimal numbers                  
    '([^a-z_$-])0x[0-9a-f]{1,10}',  '\1'||'0x',    'g'   ),

    -- Remove IN values                            
    'in\s*\([''0x,\s]*\)',          'in (...)',    'g'   )
  ;
$body$
LANGUAGE SQL;
```

Keep in mind that I extracted these regular expressions straight from pgbadger.
Any comment about how to make it quicker/better/simpler/whatever is
appreciated!

Here the associated view to group everything according to the normalized
queries:

```sql
CREATE OR REPLACE VIEW pg_stat_statements_normalized AS
SELECT userid, dbid, normalize_query(query) AS query, sum(calls) AS calls,
  sum(total_time) AS total_time, sum(rows) as rows,
  sum(shared_blks_hit) AS shared_blks_hit,
  sum(shared_blks_read) AS shared_blks_read,
  sum(shared_blks_written) AS shared_blks_written,
  sum(local_blks_hit) AS local_blks_hit,
  sum(local_blks_read) AS local_blks_read,
  sum(local_blks_written) AS local_blks_written, 
  sum(temp_blks_read) AS temp_blks_read,
  sum(temp_blks_written) AS temp_blks_written
FROM pg_stat_statements
GROUP BY 1,2,3;
```

Using this function and the view, a small `pgbench -t 30 -c 10`, gives:

```
pgbench=> SELECT round(total_time::numeric/calls, 2) AS avg_time, calls, 
  round(total_time::numeric, 2) AS total_time, rows, query 
FROM pg_stat_statements_normalized 
ORDER BY 1 DESC, 2 DESC;

 avg_time | calls | total_time | rows |                                               query                                               
----------+-------+------------+------+---------------------------------------------------------------------------------------------------
     0.05 |   187 |       9.86 |  187 | update pgbench_accounts set abalance = abalance + 0 where aid = 0;
     0.01 |   195 |       2.30 |  195 | update pgbench_branches set bbalance = bbalance + 0 where bid = 0;
     0.00 |   300 |       0.00 |    0 | begin;
     0.00 |   300 |       0.00 |    0 | end;
     0.00 |   196 |       0.00 |  196 | insert into pgbench_history (tid, bid, aid, delta, mtime) values (0, 0, 0, 0, current_timestamp);
     0.00 |   193 |       0.00 |  193 | select abalance from pgbench_accounts where aid = 0;
     0.00 |   183 |       0.26 |  183 | update pgbench_tellers set tbalance = tbalance + 0 where tid = 0;
     0.00 |     1 |       0.00 |    0 | truncate pgbench_history
```

For information, the real non-normalized `pg_stat_statement` view is 959 lines:

```
pgbench=> SELECT count(*) FROM pg_stat_statements;

 count 
-------
   959
(1 ligne)
```

Obvisouly, regular expression are not magic and this will never be as strict as
the engine itself. But at least it helps while waiting for 9.2 in production !

Do not hesitate to report me bugs and comment to improve it !

Cheers!

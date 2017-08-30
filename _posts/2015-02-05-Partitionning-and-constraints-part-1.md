---
layout: post
author: ioguix
title: Partitioning and constraints part 1 - UNIQUE
date: 2015-02-05 10:30:00 +0200
tags: [partitioning, constraints, trigger, advisory locks, postgresql]
category: postgresql
---

Partitioning in PostgreSQL has been an artisanal work for a long time now. And
despite the [current discussion](http://www.postgresql.org/message-id/20140829155607.GF7705@eldon.alvh.no-ip.org)
running since few month on PostgreSQL's hackers mailing list, it will probably
stay this way for some time again. Just because it requires a lot of brainstorm
and work.

Nowadays, I believe the current state of partitioning under PostgreSQL is quite
well documented and under control most of the time. You can find a lot of
informations about that online, starting by
[PostgreSQL documentation](http://www.postgresql.org/docs/9.4/static/ddl-partitioning.html)
itself, but about tooling as well, extension, etc.

However, there's still a dark side, not well covered or understood about
partitioning under PostgreSQL: constraints related to them. More specifically
unique constraints covering all partitions of a partitioned table and how to
refer to them from foreign keys. This series of article analysis how to
implement them by hands ourself thanks to some PostgreSQL great features,
detailing how to avoid major traps. You will see that crafting these
«constraints» wannabes requires some attention, but is definitely doable, in
a clean way.

As this subject requires quite some details and explanations, I decided to
split it in multiple articles. This first part is about creating a UNIQUE
constraint across all partitions of a table. Next one covers how to reference a
partitioned table. And maybe some other depending on the motivation,
inspiration and feedback.

{{ post.excerpt }}

## Study case

I chose to illustrate this article with a table partitioned by date range. This
is a fairly frequent practice and adapting this article to another partitioning
scheme is quite easy anyway.

Range partitioning on the PK has no challenge: each value of the PK could only
live in strictly one child, each of them enforcing the PK internally. So the
uniqueness of this PK across the partitions is already enforced by constraints
CHECK and UNIQUE of each partition. That's why my study case partition range
using a `timestampstz` column. As the CHECKs do not apply on the primary key
values, each of its values can be in any partition, which can lead to duplicate
values residing in different partitions. Exactly what we really want to avoid.

So here is the dummy schema with table `master` partitioned across 5 childs
using a date range key partitioning `ts`.

{% highlight sql %}
BEGIN;

DROP TABLE IF EXISTS master,
  child0, child1, child2, child3, child4;

CREATE TABLE master (
  id        serial PRIMARY KEY,
  dummy     int DEFAULT (random()*31449600)::int,
  comment   text,
  ts        timestamptz
);

CREATE TABLE child0 (
  PRIMARY KEY (id),
  CHECK ( ts >= '2010-01-01 00:00:00' AND ts < '2011-01-01 00:00:00' )
) INHERITS ( master );

CREATE TABLE child1 (
  PRIMARY KEY (id),
  CHECK ( ts >= '2011-01-01 00:00:00' AND ts < '2012-01-01 00:00:00' )
) INHERITS ( master );

CREATE TABLE child2 (
  PRIMARY KEY (id),
  CHECK ( ts >= '2012-01-01 00:00:00' AND ts < '2013-01-01 00:00:00' )
) INHERITS ( master );

CREATE TABLE child3 (
  PRIMARY KEY (id),
  CHECK ( ts >= '2013-01-01 00:00:00' AND ts < '2014-01-01 00:00:00' )
) INHERITS ( master );

CREATE TABLE child4 (
  PRIMARY KEY (id),
  CHECK ( ts >= '2014-01-01 00:00:00' AND ts < '2015-01-01 00:00:00' )
) INHERITS ( master );

CREATE INDEX ON child0 (ts);
CREATE INDEX ON child1 (ts);
CREATE INDEX ON child2 (ts);
CREATE INDEX ON child3 (ts);
CREATE INDEX ON child4 (ts);

ALTER TABLE child0 ALTER ts SET
  DEFAULT TIMESTAMPTZ '2010-01-01 00:00:00' + (random()*31449600)::int * INTERVAL '1s';
ALTER TABLE child1 ALTER ts SET
  DEFAULT TIMESTAMPTZ '2011-01-01 00:00:00' + (random()*31449600)::int * INTERVAL '1s';
ALTER TABLE child2 ALTER ts SET
  DEFAULT TIMESTAMPTZ '2012-01-01 00:00:00' + (random()*31449600)::int * INTERVAL '1s';
ALTER TABLE child3 ALTER ts SET
  DEFAULT TIMESTAMPTZ '2013-01-01 00:00:00' + (random()*31449600)::int * INTERVAL '1s';
ALTER TABLE child4 ALTER ts SET
  DEFAULT TIMESTAMPTZ '2014-01-01 00:00:00' + (random()*31449600)::int * INTERVAL '1s';

COMMIT;
{% endhighlight %}

The `SET DEFAULT` are only there to keep other commands simple to read. Note
that I do not create the trigger on INSERT and UPDATE on the master table. This
is out of the scope of this article, will not be needed and add no challenge to
the subject.

## The naive solution

Of course, the whole trick revolves around triggers. We have to check the
uniqueness of a PK value across all partitions after any INSERT or UPDATE on
any of them, for each rows. Let's dive in and get wet with a first naive
version of such a trigger:

{% highlight sql %}
CREATE OR REPLACE FUNCTION master_id_pkey() 
  RETURNS trigger 
  LANGUAGE plpgsql 
AS $$
BEGIN
  IF count(1) > 1 FROM master WHERE id = NEW.id THEN
    RAISE EXCEPTION 'duplicate key value violates unique constraint "%" ON "%"', 
      TG_NAME, TG_TABLE_NAME 
      USING DETAIL = format('Key (id)=(%s) already exists.', NEW.id);
  END IF;

  RETURN NULL;
END
$$;

CREATE TRIGGER children_id_pkey AFTER INSERT OR UPDATE ON master
     FOR EACH ROW EXECUTE PROCEDURE public.master_id_pkey();
CREATE TRIGGER children_id_pkey AFTER INSERT OR UPDATE ON child0
     FOR EACH ROW EXECUTE PROCEDURE public.master_id_pkey();
CREATE TRIGGER children_id_pkey AFTER INSERT OR UPDATE ON child1
     FOR EACH ROW EXECUTE PROCEDURE public.master_id_pkey();
CREATE TRIGGER children_id_pkey AFTER INSERT OR UPDATE ON child2
     FOR EACH ROW EXECUTE PROCEDURE public.master_id_pkey();
CREATE TRIGGER children_id_pkey AFTER INSERT OR UPDATE ON child3
     FOR EACH ROW EXECUTE PROCEDURE public.master_id_pkey();
CREATE TRIGGER children_id_pkey AFTER INSERT OR UPDATE ON child4
     FOR EACH ROW EXECUTE PROCEDURE public.master_id_pkey();
{% endhighlight %}

Obviously, each partition need the trigger to check that the PK value it is
about to write does not already exist in one of its siblings. The trigger
function itself is quite easy to understand: if we find a row with the same
value as the PK, raise an exception. Tests sounds promising:

{% highlight psql %}
=# INSERT INTO child0 (comment) VALUES ('test 1');
=# INSERT INTO child1 (comment) VALUES ('test 2');
=# INSERT INTO child2 (comment) VALUES ('test 3');
=# INSERT INTO child3 (comment) VALUES ('test 4');
=# INSERT INTO child4 (comment) VALUES ('test 5');
=# SELECT tableoid::regclass, * FROM master;
 tableoid | id |  dummy   | comment |           ts           
----------+----+----------+---------+------------------------
 child0   |  1 | 22810434 | test 1  | 2010-07-29 02:49:24+02
 child1   |  2 | 18384970 | test 2  | 2011-01-28 02:57:00+01
 child2   |  3 | 10707988 | test 3  | 2012-05-17 04:58:36+02
 child3   |  4 | 15801904 | test 4  | 2013-08-21 10:31:04+02
 child4   |  5 | 14906458 | test 5  | 2014-10-16 00:09:58+02
(5 rows)


=# BEGIN ;
=# INSERT INTO child0 (id, comment) VALUES (5, 'test 6');
ERROR:  duplicate key value violates unique constraint "children_id_pkey" ON "child0"
DETAIL:  Key (id)=(5) already exists.
{% endhighlight %}

OK, it works like expected. But there is two big issues with this situation.
The first one is that a race condition involving two transactions or more is
able to break our home made unique constraint:

{% highlight psql %}
session 1=# BEGIN;
BEGIN

session 2=# BEGIN;
session 2=# INSERT INTO child0 (comment) VALUES ('test 6') RETURNING id;
 id
----
  6

session 1=# INSERT INTO child1 (id, comment) VALUES (6, 'test 7');
session 1=# COMMIT ;

session 2=# COMMIT ;
session 2=# SELECT tableoid::regclass, * FROM master WHERE id=6;
 tableoid | id |  dummy   | comment |           ts           
----------+----+----------+---------+------------------------
 child0   |  6 | 28510860 | test 6  | 2010-01-08 17:36:39+01
 child1   |  6 |  2188136 | test 7  | 2011-07-15 07:13:59+02
{% endhighlight %}

The second issue is that real constraints can be deferred, which means
constraints are disabled during a transaction and enforced on user request and
at the end of the transaction by default. In other words, using deferred
constraints allows you to violate them temporarilly during a transaction as far
as everything is respected at the end. For more information about this
mechanism, see the
[SET CONSTAINTS](http://www.postgresql.org/docs/9.4/static/sql-set-constraints.html),
[CREATE TABLE](http://www.postgresql.org/docs/9.4/static/sql-createtable.html)
and... the
[CREATE TRIGGER](http://www.postgresql.org/docs/9.4/static/sql-createtrigger.html)
pages.

Yes, documentation says triggers can be deferred when defined as
`CONSTRAINT TRIGGER`. So we can solve this issue by recreating our triggers:

{% highlight sql %}
DROP TRIGGER IF EXISTS children_id_pkey ON master;
DROP TRIGGER IF EXISTS children_id_pkey ON child0;
DROP TRIGGER IF EXISTS children_id_pkey ON child1;
DROP TRIGGER IF EXISTS children_id_pkey ON child2;
DROP TRIGGER IF EXISTS children_id_pkey ON child3;
DROP TRIGGER IF EXISTS children_id_pkey ON child4;
CREATE CONSTRAINT TRIGGER children_id_pkey AFTER INSERT OR UPDATE ON master
     DEFERRABLE INITIALLY IMMEDIATE FOR EACH ROW EXECUTE PROCEDURE public.master_id_pkey();
CREATE CONSTRAINT TRIGGER children_id_pkey AFTER INSERT OR UPDATE ON child0
     DEFERRABLE INITIALLY IMMEDIATE FOR EACH ROW EXECUTE PROCEDURE public.master_id_pkey();
CREATE CONSTRAINT TRIGGER children_id_pkey AFTER INSERT OR UPDATE ON child1
     DEFERRABLE INITIALLY IMMEDIATE FOR EACH ROW EXECUTE PROCEDURE public.master_id_pkey();
CREATE CONSTRAINT TRIGGER children_id_pkey AFTER INSERT OR UPDATE ON child2
     DEFERRABLE INITIALLY IMMEDIATE FOR EACH ROW EXECUTE PROCEDURE public.master_id_pkey();
CREATE CONSTRAINT TRIGGER children_id_pkey AFTER INSERT OR UPDATE ON child3
     DEFERRABLE INITIALLY IMMEDIATE FOR EACH ROW EXECUTE PROCEDURE public.master_id_pkey();
CREATE CONSTRAINT TRIGGER children_id_pkey AFTER INSERT OR UPDATE ON child4
     DEFERRABLE INITIALLY IMMEDIATE FOR EACH ROW EXECUTE PROCEDURE public.master_id_pkey();
{% endhighlight %}

The `INITIALLY IMMEDIATE` means the trigger constraint will be executed right
after the related statement. The opposite `DEFERRED` behavior fire the trigger
at the very end of the transaction unless the user decide to `SET CONSTRAINTS {
ALL | name [, ...] } IMMEDIATE` somewhere during the transaction.

## Defering the trigger to avoid the race condition ?

Now, if you step back a second to look at what we have, you might wonder if
forcing our constraints triggers to be `DEFERRABLE INITIALLY DEFERRED` would
solve the race condition. As constraints are checked at the very end of the
transaction, maybe this would work by kind of serializing each transaction and
their constraints? Short answer is: no.

For one, deferred constraints comes with a cost in performance we might not
want to pay at each transaction. But most importantly, if you declared your
trigger as deferrable, one could set it to IMMEDIATE, even if it is set as
INITIALLY DEFERRED. So this is definitely not a viable solution. But anyway,
occulting this for the purpose of the study, does it work?

Again, no. Even if it solves the "human timing race condition", there's another
very small window where another race condition is possible in the core of
PostgreSQL, when multiple transactions do not conflict and get committed all
together at the exact same time. This idea itself sounds suspicious anyway, too
fragile. If there is no good ol'locks floating around, there's a race condition
close enough to break things. It is pretty easy to prove with the following
bash loop hammering each partitions with 100 INSERTs with colliding values as
PK. Note that the triggers has been altered to `INITIALLY DEFERRED`:

{% highlight console %}
$ psql -c '\d child*' part | grep children_id_pkey
    children_id_pkey AFTER INSERT OR UPDATE ON child0 DEFERRABLE INITIALLY DEFERRED FOR EACH ROW EXECUTE PROCEDURE master_id_pkey()
    children_id_pkey AFTER INSERT OR UPDATE ON child1 DEFERRABLE INITIALLY DEFERRED FOR EACH ROW EXECUTE PROCEDURE master_id_pkey()
    children_id_pkey AFTER INSERT OR UPDATE ON child2 DEFERRABLE INITIALLY DEFERRED FOR EACH ROW EXECUTE PROCEDURE master_id_pkey()
    children_id_pkey AFTER INSERT OR UPDATE ON child3 DEFERRABLE INITIALLY DEFERRED FOR EACH ROW EXECUTE PROCEDURE master_id_pkey()
    children_id_pkey AFTER INSERT OR UPDATE ON child4 DEFERRABLE INITIALLY DEFERRED FOR EACH ROW EXECUTE PROCEDURE master_id_pkey()

$ psql -c 'truncate master cascade' part

$ for i in {1..100}; do
>   psql -c "INSERT INTO child0 (id, comment) SELECT count(*)+1, 'duplicated ?' FROM master" part &
>   psql -c "INSERT INTO child1 (id, comment) SELECT count(*)+1, 'duplicated ?' FROM master" part &
>   psql -c "INSERT INTO child2 (id, comment) SELECT count(*)+1, 'duplicated ?' FROM master" part &
>   psql -c "INSERT INTO child3 (id, comment) SELECT count(*)+1, 'duplicated ?' FROM master" part &
>   psql -c "INSERT INTO child4 (id, comment) SELECT count(*)+1, 'duplicated ?' FROM master" part &
> done &> /dev/null && wait

$ cat <<EOQ | psql part
> SELECT count(1), appears, total FROM (
>   SELECT id, count(1) AS appears, sum(count(*)) over () AS total
>   FROM master
>   GROUP BY id
> ) t 
> GROUP BY 2,3 ORDER BY appears
> EOQ
 count | appears | total 
-------+---------+-------
   149 |       1 |   209
    23 |       2 |   209
     3 |       3 |   209
     1 |       5 |   209
{% endhighlight %}

Well, that's pretty bad, we have a bunch of duplicated key. 23 of them appear
in two different partitions, three others in three different partitions and
even one in all of them! I could find duplicates like that each time I ran this
scenario. Note that on 500 inserts, only 209 survived in total. That makes 291
exceptions raised out of 324 expected, counting the duplicated keys that were
not caught.

## Isolation level?

Well, last chance. If this many transactions were committed in the exact same
time, maybe we can force them to serialize with isolation level `SERIALIZABLE`?

{% highlight sql %}
ALTER DATABASE part SET default_transaction_isolation TO SERIALIZABLE
{% endhighlight %}

After applying the preceding query, I re-ran the same scenario as the previous
test: only 76 rows survived out of the 500 INSERTs, all of them unique. At
last! Ok, this reflects what we had in mind previously, but we had to force
PostgreSQL to *really* serialize transactions. Any other isolation level will
just fail. And by the way, this works with IMMEDIATE and DEFERRED triggers as
transactions are virtually serialized or rollback'ed. Log file confirms a lot
of serialization conflicts were raised, grep'ing the log file shows 415
serialization exceptions and only 9 from our trigger:

{% highlight text %}
ERROR:  could not serialize access due to read/write dependencies among transactions
DETAIL:  Reason code: Canceled on identification as a pivot, during conflict in checking.
HINT:  The transaction might succeed if retried.
STATEMENT:  INSERT INTO child3 (id, comment) SELECT count(*)+1, 'duplicated ?' FROM master
{% endhighlight %}

This solution work, but having to stay in SERIALIZABLE mode to achieve our goal
is a heavy constraint to carry. Moreover, we have the same problem than with
DEFERRED triggers: as a simple user can change its isolation level, any bug in
the application or not informed user can lead to scenarios with silent
duplications. Fortunately, another simpler and safer solution exist.

## Real solution: adding locks

The `SERIALIZABLE` solution works because to emulate serial transaction
execution for all transactions, it takes __predicate locks__ behind the scene
to detect serialization anomalies. What about taking care of this ourselves? We
are used to locks, we know they work fine.

The best solution sounds to acquire a lock before being able to write out the
value. This actually boil down to forcing conflicting transactions on a lock to
serialize themselves, instead of having the engine do all the works for
everyone. The question now is «how can we hold a lock on something that
doesn't exists yet?». The answer is:
[Advisory Locks](http://www.postgresql.org/docs/9.4/static/explicit-locking.html#ADVISORY-LOCKS).
Advisory locks offers to applications a lock mechanism and manager on arbitrary
integer values. It does not applies on real objects, transaction or rows. As
the documentation says: «It is up to the application to use them correctly».

The idea now is simply to acquire an advisory lock on the same value as NEW.id
in the trigger function. It should do the trick, cleanly, safely:

{% highlight sql %}
CREATE OR REPLACE FUNCTION public.master_id_pkey()
 RETURNS trigger
 LANGUAGE plpgsql
AS $function$
BEGIN
  PERFORM pg_advisory_xact_lock(NEW.id);

  IF count(1) > 1 FROM master WHERE id = NEW.id THEN
     RAISE EXCEPTION 'duplicate key value violates unique constraint "%" ON "%"', 
      TG_NAME, TG_TABLE_NAME 
      USING DETAIL = format('Key (id)=(%s) already exists.', NEW.id);
  END IF;

  RETURN NULL;
END
$function$;
{% endhighlight %}

And with this version of `master_id_pkey()`, in "read committed" isolation
level, here is result of the same scenario as in the previous chapter,
executing 500 INSERTs concurrently with conflicting keys:

{% highlight console %}
$ psql -f /tmp/count_duplicated_id.sql part
 count | appears | total 
-------+---------+-------
    85 |       1 |    85 
(1 row)
{% endhighlight %}

Sounds good. What about a small pgbench scenario?

{% highlight console %}
$ cat /tmp/scenario_id.sql 
\setrandom part 0 4
DO $func$ BEGIN EXECUTE format('INSERT INTO child%s (id, comment) SELECT count(*)+1, $1 FROM master', :part) USING 'duplicated ?'; EXCEPTION WHEN OTHERS THEN RAISE LOG 'Duplicate exception caught!'; END $func$;

$ psql -c 'truncate master cascade' part

$ pgbench -n -f /tmp/scenario_id.sql -c 5 -T 300 part
transaction type: Custom query
scaling factor: 1
query mode: simple
number of clients: 5
number of threads: 1
duration: 300 s
number of transactions actually processed: 130908
latency average: 11.458 ms
tps = 436.338755 (including connections establishing)
tps = 436.354969 (excluding connections establishing)

$ psql -f /tmp/count_duplicated_id.sql part
 count | appears | total 
-------+---------+-------
 48351 |       1 | 48351

$ grep -c "LOG:  Duplicate" $LOGFILE
82557
{% endhighlight %}

After this 5 minute run with 5 workers inserting as fast as they can highly
conflicting data, we have 48,351 rows in the partitions, 82,557 conflicting
rows were rejected and not a single duplicate in the table.

I couldn't find any duplicated value after stressing this solution. Whatever
the number of queries for each parallel sessions working, whatever the pgbench
scenario, I had no unique violation across partitions, as expected. This work
in any transaction isolation level and user can not turn this off by mistake.
This is *safe*...

...Well, as far as a superuser or the owner of the table do not disable the
trigger on the table, obviously. But hey, they can drop the unique constraint
on a normal table as well, right?

Wow, at last, finished. What? No? I can hear you thinking it only applies on
integers. OK, bonus.

## Supporting other types

Supporting unique constraint on integers was straightforward using advisory
locks. But how can this applies to other types? Like text for instance ? Easy:
hash it[^1]! For the purpose of this last chapter, lets add a unique constraint
on `comment`:

{% highlight sql %}
CREATE OR REPLACE FUNCTION public.master_comment_unq()
 RETURNS trigger
 LANGUAGE plpgsql
AS $function$
BEGIN
  PERFORM pg_advisory_xact_lock(hashtext(NEW.comment));

  IF count(1) > 1 FROM master WHERE comment = NEW.comment THEN
    RAISE EXCEPTION 'duplicate key value violates unique constraint "%" ON "%"', 
      TG_NAME, TG_TABLE_NAME
      USING DETAIL = format('Key (comment)=(%L) already exists.', NEW.comment);
  END IF;

  RETURN NULL;
END
$function$;

CREATE CONSTRAINT TRIGGER children_comment_unq AFTER INSERT OR UPDATE ON master
  DEFERRABLE INITIALLY IMMEDIATE FOR EACH ROW EXECUTE PROCEDURE public.master_comment_unq();
-- [...]
CREATE CONSTRAINT TRIGGER children_comment_unq AFTER INSERT OR UPDATE ON child4
  DEFERRABLE INITIALLY IMMEDIATE FOR EACH ROW EXECUTE PROCEDURE public.master_comment_unq();
{% endhighlight %}

If you followed this far, no need to play the "find the error game" to identify
what's the most important change here. The lock is taken on the result of the
simple text to integer hash function `hashtext`, already provided in
PostgreSQL's core.

Ok, I can hear optimization freaks crying. Theoretically, two *different*
strings can collide. But this hash function is supposed to compute uniform
results amongst 4 billion possible values. I can live with the probability of
two *concurrent* writes involving two different strings colliding here. The "1
case" out of 4 billion is already enough for me, but these colliding strings
has to show up at the *exact same time* (at least in the same bunch of few
milliseconds). And even if you are unlucky enough to experience this, these two
transactions will just be serialized, not a big deal.

And if you are really not comfortable with this, you understood the trick here
anyway: find a way to hold a lock somewhere to avoid concurrency. Use some
other hashing function, create an extension with its own lock machinery in
memory, write in an unlogged table (erk), whatever you want.

Time to test now.

{% highlight console %}
$ cat /tmp/scenario_comment.sql 
\setrandom part 0 4
DO $func$ BEGIN EXECUTE format('INSERT INTO child%s (comment) SELECT ''duplicated ''||count(1) from master', :part); EXCEPTION WHEN OTHERS THEN RAISE LOG 'Duplicate exception caught!'; END $func$;

$ psql -c 'truncate master cascade' part

$ pgbench -n -f /tmp/scenario_comment.sql -c 5 -T 300 part
transaction type: Custom query
scaling factor: 1
query mode: simple
number of clients: 5
number of threads: 1
duration: 300 s
number of transactions actually processed: 93902
latency average: 15.974 ms
tps = 312.971273 (including connections establishing)
tps = 312.987557 (excluding connections establishing)

$ cat <<EOQ | psql part
> SELECT count(1), appears, total FROM (
>   SELECT comment, count(1) AS appears, sum(count(*)) over () AS total
>   FROM master
>   GROUP BY comment
> ) t 
> GROUP BY 2,3 ORDER BY appears
> EOQ
 count | appears | total 
-------+---------+-------
 29785 |       1 | 29785
{% endhighlight %}

Wow (again), only 29,785 rows out of 93,902 transactions escaped of that
intense-colliding scenario. And we only find unique values across all
partitions, as expected. Aaaaand grep-ing from the log file, I can find 64,117
rejected rows...

## Conclusion

Such a long way already. Thank you for reading so far. At first I thought I
could write about unique and foreign keys in the same article, but look at what
we already covered...We talked about constraint triggers, race conditions,
isolation level, advisory locks and hashing... few!

I do realize the solution provided here requires some skills and attention.
This is not all magic and easy to play with. As long as this feature is not
handled directly by the core, partitioning will require people to craft their
tools themselves. In the meantime, it is a nice subject to learn more about
these concepts, your favorite RDBMS and play with it.

I think this way of guaranteeing unicity over several partitions is
bulletproof. If you think you found a loophole, please send me some feedback,
I'll be pleased to learn about them.

And don't forget, we are not done! Lot of fun with foreign keys in the next
part! Stay tuned!

<hr style="width: 100px; text-align: left" />

[^1]: No data has been harmed during this test.

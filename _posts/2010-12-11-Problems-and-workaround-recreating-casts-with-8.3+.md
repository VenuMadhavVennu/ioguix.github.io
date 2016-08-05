---
layout: post
author: ioguix
title: Problems and workaround recreating implicit casts using 8.3+
date: 2010-12-11 16:00 +0100
tags: [postgresql, casts]
category: postgresql
---

There's a very frequent issue when upgrading to version 8.3 and bellow: the
removal of some implicit casts from text types to time or numerical ones in
8.3.  The only clean solution here is to fix the application itself, period.
However, for those that cannot afford quickly such a hard work, the popular
workaround is to recreate these implicit casts, but it suffer from a side
effect. About a year ago, I found another quick-n-dirty fix for a customer.

{{ post.excerpt }}

Here is the problem:

```sql
casts=# CREATE TABLE test AS SELECT generate_series(1,10) as id;
SELECT
casts=# SELECT id, 'value = ' || id FROM test WHERE id = '5'::text;
ERROR:  operator does not exist: integer = text
LINE 1 : SELECT id, 'value = ' || id FROM test WHERE id = '5'::text;
                                                         ^
TIPS : No operator matches the given name and argument type(s). You might need to add explicit type casts.
```

The very well known solution is to recreate some of these implicit casts that
were removed in 8.3. Peter Eisentraut blogged about that, you'll find his SQL
script
[here](http://petereisentraut.blogspot.com/2008/03/readding-implicit-casts-in-postgresql.html).

However, as some users noticed in the comments, there is a side effect bug with
this solution: it breaks the concatenation operator.

```sql
casts=# BEGIN ;
BEGIN
casts=# \i /tmp/implicit_casts.sql
CREATE FUNCTION
CREATE CAST
-- [...]
CREATE FUNCTION
CREATE CAST
casts=# SELECT id, 'value = ' || id FROM test WHERE id = '5'::text;
ERROR:  operator is not unique: unknown || integer
LINE 1 : SELECT id, 'value = ' || id FROM test WHERE id = '5'::text;
                                ^
TIPS : Could not choose a best candidate operator. You might need to add explicit type casts.
casts=# ROLLBACK ;
ROLLBACK
```

From here, the solution could be to cast one of the operand:

```sql
casts=# SELECT id, 'value = ' || id::text FROM test WHERE id = '5'::text;
  5 | value = 5
```

But then, we are back to the application fix where it might worth spending more
time fixing things in the good way.

There is another solution: creating missing operators instead of implicit
casts. You will find a sql file with a lot of those operators under the
following link: [8.3 operator workaround.sql](https://gist.github.com/ioguix/4dd187986c4a1b7e1160).
Here is a sample for text to integer comparison:

```sql
CREATE FUNCTION pg_catalog.texteqint(text, integer) RETURNS BOOLEAN STRICT IMMUTABLE LANGUAGE SQL AS $$SELECT textin(int4out($2)) = $1;$$;
CREATE FUNCTION pg_catalog.inteqtext(integer, text) RETURNS BOOLEAN STRICT IMMUTABLE LANGUAGE SQL AS $$SELECT textin(int4out($1)) = $2;$$;
CREATE OPERATOR pg_catalog.= ( PROCEDURE = pg_catalog.texteqint, LEFTARG=text, RIGHTARG=integer, COMMUTATOR=OPERATOR(pg_catalog.=));
CREATE OPERATOR pg_catalog.= ( PROCEDURE = pg_catalog.inteqtext, LEFTARG=integer, RIGHTARG=text, COMMUTATOR=OPERATOR(pg_catalog.=));
```

Using this operator instead of implicit cast, the previous test shows:

```sql
casts=# BEGIN ;
BEGIN
casts=# \i '/tmp/8.3 operator workaround.sql'
CREATE FUNCTION
-- [...]
CREATE FUNCTION
CREATE OPERATOR
-- [...]
CREATE OPERATOR
casts=# SELECT id, 'value = ' || id FROM test WHERE id = '5'::text;
  5 | value = 5

casts=# -- what, you don't trust me :) ?
casts=# ROLLBACK ;
ROLLBACK
casts=# SELECT id, 'value = ' || id FROM test WHERE id = '5'::text;
ERROR:  operator does not exist: integer = text
LINE 1 : SELECT id, 'value = ' || id FROM test WHERE id = '5'::text;
                                                         ^
TIPS : No operator matches the given name and argument type(s). You might need to add explicit type casts.
```

Same advice from Peter here: if possible, only create the operators you need to
fix your application!

So far, I only had __one__ positive feedback about this workaround about a year
ago, and I don't consider this is enough to actually claim it is a safe
solution. So please, comments, tests and reports are welcome!

Again, keep in mind that the only clean way is fix your application if you hit
this problem!

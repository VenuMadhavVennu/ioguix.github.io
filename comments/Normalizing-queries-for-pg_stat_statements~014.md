---
title: Normalizing queries for pg_stat_statements < 9.2
author: Peter Geoghegan
date: 2012-08-06 22:28:43 +0200
---
When I first started reading this post, I assumed that the idea was to normalise the query text on the fly within the executor hooks that pg_stat_statements uses. While that might be quite inefficient, it would at least have the advantage of not eating an entry from the shared hashtable for every set of constants for every query seen.  The hashtable will constantly have entries evicted, so you're only going to have a very limited window on the queries executed against the database. Look at the "calls" figure for each query after executing pgbench in your example.

Why not somehow compile the regular expression ahead of time?

FYI, it isn't that hard to backport the changes to the core system that make the new pg_stat_statements work (mostly it's just that there are new hooks added). You'd have to fix pg_stat_statements to work with the 9.1 representation of the query tree (the code won't compile, because the representation of CTAS changed, and perhaps other small things too). The "jumbling" code you'd have to modify is fairly straightforward though.

---
layout: post
author: ioguix
title: New repository for bloat estimation queries
date: 2015-02-26 00:05:00 +0200
tags: [bloat, postgresql]
category: postgresql
---

##New repository

It's been almost a year now that I wrote the first version of the btree bloat
estimation query.  Then, came the first fixes, the bloat estimation queries for
tables, more fixes, and so on.  Maintaining these queries as gists on github
was quite difficult and lack some features: no documented history, multiple
links, no doc, impossible to fork, etc.

So I decided to move everything to a git repository you can fork right away:
[http://github.com/ioguix/pgsql-bloat-estimation].
There's already 10 commits for improvements and bug fixes.

{{ post.excerpt }}

Do not hesitate to fork this repo, play with the queries, test them or make
pull requests.  Another way to help is to discuss your results or report bugs
by opening [issues](https://github.com/ioguix/pgsql-bloat-estimation/issues).
This can lead to bug fixes or the creation of a FAQ.


##Changelog

Here is a quick changelog since my
[last post about bloat]( {% post_url 2014-11-03-Btree-bloat-query-part-4 %}):

* support for fillfactor!  Previous versions of the queries were considering
  any extra space as bloat, even the fillfactor.  Now, the bloat is reported
  without it.  So a btree with the default fillfactor and no bloat will report a
  bloat of 0%, not 10%.
* fix bad tuple header size for tables under 8.0, 8.1 or 8.2.
* fix bad header size computation for varlena types.
* fix illegal division by 0 for the btrees.
* added some documentation!  See
  [README.md](https://github.com/ioguix/pgsql-bloat-estimation/blob/master/README.md).

In conclusion, do not hesitate to use this queries in your projects, contribute
to them and make some feedback!

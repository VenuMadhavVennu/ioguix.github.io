---
title: More work and thoughts on index bloat estimation query
author: Jehan-Guillaume (ioguix) de Rorthais
date: 2014-06-25 18:35 +0200
---
Hi Clemens,

> the bytes output should be converted into human readable format (mb,gb,etc)

The main purpose of this query is supervision, so I need raw values, but adding some more fields with human values is clearly useful.

> internal postgresql tables shouldn't be shown

I decided to keep all the objects found in the database, even system ones. They can be removed easily by filtering out on the schemaname column if needed

> it should be sorted by most bloat first

Agree with that, easy to add.

> See this index bloat query: [https://gist.github.com/gullevek/32881d6b4c5b1ed0135c]() (original here: [https://gist.github.com/mbanck/9976015]())

I know this query, it has been written by Josh Berkus based on my previous post. As I commented in his blog, I don't want to break compatibility with older release of PostgreSQL by using CTEs, just for human readability. I need it to monitor old releases (sadly, like 8.2, but even a 7.4!).

I'll update the query in my gist and this blog post with your comments.

Cheers,

---
title: Playing with indexes and better bloat estimate
author: ioguix
date: 2014-04-03 15:43:33 +0200
---
Josh,

About `index_tuple_hdr`, I just realize I used it as a field name AND a psql variable, my bad. See the code bellow this comment in the query:

`/* per tuple header: add index_attribute_bm if some cols are null-able */`

So if you are referring to `index_tuple_hdr` at lines 11, 13 and 14, those are referring to the computed field, so no `:`, it would produce bad stats with indexes on NULLable cols.

I fixed to query to remove this confusion.

Good catch about the `\SET`! It appears there was some magic in my old blog system transforming it in uppercase. I'll not investigate further as I moved to jekyll...

Thanks!

---
title: Playing with indexes and better bloat estimate
author: Josh Berkus
date: 2014-04-01 02:10:31 +0200
---
Actually, there's some bugs in the query version as you've written it above:

- It's `\set`, not `\SET`
- `index_tuple_hdr` is missing the `:` in two places.


Thanks!

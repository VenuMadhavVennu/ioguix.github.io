---
title: Normalizing queries for pg_stat_statements < 9.2
author: depesz
date: 2012-08-06 20:53:04 +0200
---
This is not the first place I see it, so I have to ask: why "[\t\s\r\n]+" and not simply \s+ ? \s class already includes \t, \r, \n, and most likely also \f and \v.

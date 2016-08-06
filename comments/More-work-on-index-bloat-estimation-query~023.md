---
title: More work and thoughts on index bloat estimation query
author: Clemens Schwaighofer
date: 2014-06-25 11:52:08 +0900
---
Interesting query, but there are several things that should be updated.

* the bytes output should be converted into human readable format (mb,gb,etc)
* internal postgresql tables shouldn't be shown
* it should be sorted by most bloat first

See this index bloat query: [https://gist.github.com/gullevek/32881d6b4c5b1ed0135c]() (original here: [https://gist.github.com/mbanck/9976015]())

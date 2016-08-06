---
title: Playing with indexes and better bloat estimate
author: Josh Berkus
date: 2014-04-01 01:58:01 +0200
---
Thanks for posting this!  I've known for a while that the index bloat calculation was faulty, but haven't quite known what was wrong with it.

I'm working on a 2nd bloat calculation involving dead row counts -- just for tables,though.


---
layout: post
author: ioguix
title: GSoC project ideas for phpPgAdmin
date: 2011-03-28 21:25 +0200
tags: [phppgadmin, gsoc, postgresql]
category: postgresql
---
GSoC season started !

PostgreSQL has been accepted this year again as project organization.  Big
thanks to Selena and others to handle this for the community!  You can find the
PostgreSQL GSoC 2011 page "here":http://wiki.postgresql.org/wiki/GSoC_2011.  As
far as I know, student projects submissions starts today.

{{ post.excerpt }}

phpPgAdmin is a sub-project of the PostgreSQL organization and already had 3 or
4 projects accepted in past GSoC programs. 

I was mentoring for the first time one project for phpPgAdmin last year and
enjoyed it.  Leonardo Augusto Sápiras was the student that worked on this
project and did a good job.  He was working hard, learning how to use
community's tools, how to communicate with us, discussing issues, code and
managed to finish his project in time.  All his work was committed during or
soon after the end of the GSoC...everything while still being at school as a
Brazilian student!

Good news is that he's motivated this year again and is currently writing a
proposal to add a long time wanted feature: a proper plugin architecture.  It
might be a good subject for another blog post later.  Hopefully his proposal
will be accepted.

So, here are some more ideas for PPA:

* support PostgreSQL 9.1
* switch PPA to UTF-8 only
* graphical explain: kind of a merge between pgAdmin one and the excellent
  [explain.depesz.com](http://explain.depesz.com/).  But to be honest, as I
  told to Leonardo, I would like to keep it for me
* add support for multi-edit/delete/add data from tables
* drop adodb keeping the compatibility.  A lots of advantages here: remove
  dependency, ability to use some postgresql-only php function, lighter
  low-level db access layer, last but not least: ability to keep a full history
  of PPA queries, downloading it etc
* support for showing/editing database/user/database+user/function specific
  configuration parameters
* cleanup / improve xhtml code for better themes, accessibility and add some
  more theme

Plus, check our TODO file for a bunch of pending
[TODOs](https://github.com/phppgadmin/phppgadmin/raw/master/TODO) or our
[feature request](https://sourceforge.net/tracker/?group_id=37132&atid=418983)
 list on sf.net for more ideas!

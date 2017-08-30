---
layout: post
author: ioguix
title: PAF 2.2 rc1 released
date: 2017-08-30 14:00:00 +0200
tags: [postgresql, ha, pacemaker]
category: postgresql
---

The first release candidate of the PAF resource agent for Pacemaker has been
released yesterday.

You'll find the tarball, packages and details there:
[https://github.com/dalibo/PAF/releases/tag/v2.2_rc1]()

The changelog since 2.1 includes:

* new: support PostgreSQL 10
* new: add the maxlag parameter to exclude lagging slaves from promotion, 
  Thomas Reiss
* new: support for multiple pgsqlms resources in the same cluster
* new: provide comprehensive error messages to crm_mon
* fix: follow the resource agent man page naming policy and section
* fix: add documentation to the pgsqlms man page
* fix: do not rely on crm_failcount, suggested on the clusterlabs lists
* misc: improve the RPM packaging
* misc: check Pacemaker compatibility and resource setup
* misc: improve the election process by including timelines comparison
* misc: various code cleanup, factorization and module improvement

But I have some other great news as well. First, various documentation pages
has been updated:
* the Debian cookbook catched up with the CentOS one
* quick starts has been updated and improved
* a new quick start for Debian 9 has been added (spoiler: crmsh is much better
  supported under Debian)

See [http://dalibo.github.io/PAF/documentation.html]().

Second, PAF packages are now available in the yum PGDG repository thanks to
Devrim! It currently delivers 2.1 or 1.1 depending on your distro, but will
delivers 2.2 as soon as it is officially released.

Moreover, 2.2 is currently in the pgdg-testing repositories of
[http://apt.postgresql.org](). It will hit the stable ones after its official
release as well. Thank you Adrian Vondendriesch for your help and patches!

Of course, any kind of feedback or contribution is welcomed.

Cheers!

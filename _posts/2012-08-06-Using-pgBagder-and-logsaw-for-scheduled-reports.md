---
layout: post
author: ioguix
title: Using pgBagder and logsaw for scheduled reports
date: 2012-08-06 15:37 +0200
tags: [postgresql, pgbadger, logsaw]
category: postgresql
---
Hey,

While waiting for next version of pgBadger, here is a tip to create scheduled
pgBadger report.  For this demo, I'll suppose:
* we have PostgreSQL's log files in `/var/log/pgsql`
* we want to produce a weekly report using pgBadger with the "postgres" system
  user... 
* ...so we keep at least 8 days of log

You will need [pgbadger](https://github.com/dalibo/pgbadger/downloads) and
[logsaw](https://github.com/dalibo/logsaw/downloads).  Both tools are under BSD
license and pure perl script with no dependencies.

"logsaw" is a tool aimed to parse log files, looking for some regexp-matching
lines, printing them to standard output and remembering where it stop last
time.  At start, it searches for the last line it parsed during its last call,
and starts working from there.  Yes, for those familiar with "tail_n_mail", it
does the same thing, but without the mail and report processing part.
Moreover, it supports rotation and compression of log files (not sure about
tail_n_mail).  Thanks to this tool, we'll be able to create new reports from
where the last one finished!

We need to create a simple configuration file for logsaw:

```bash
cat <<EOF > ~postgres/.logsaw
LOGDIR=/var/log/pgsql/
EOF
```

There's three more optional parameters in this configuration file you might
want to know:

* if you want to process some particular files, you can use the
  `LOGFILES` parameter, a regular expression to filter files in the
  `LOGDIR` folder. When not defined, the default empty string means:
  «take all files in the folder».
* if your log files are compressed, you can use the `PAGER` parameter which
  should be a command that uncompress your log files to standard output, eg.
  `PAGER=zcat -f`. Note that if available, logsaw will use silently
  IO::Zlib to read your compressed files.
* you can filter extracted lines from log files using the `REGEX` parameters.
  You can add as many `REGEX` parameter than needed, each of them must be a
  regular expression. When not defined, the default empty `REGEX` array means:
  «match all the lines».
* each time you call logsaw, it saves its states to your configuration file,
adding parameters `FILEID` and `OFFSET`. 

That's it for logsaw. See its README files for more details, options and sample
configuration file. 

Now, the command line to create the report using pgbadger:

```
logsaw | pgbadger -g -o /path/to/report-$(date +%Y%m%d).html -
```

You might want to add the <code>-f</code> option to pgbadger if it doesn't
guess the log format itself (stderr or syslog or csv).

About creating reports on a weekly basis, let's say every sunday at 2:10am,
using crontab:

```bash
10 2 1-7 * 7  logsaw | pgbadger -g -o /path/to/report-$(date +\%Y\%m\%d).html -
```

Here you go, enjoy :)

That's why I like UNIX style and spirit commands: simple single-task but
powerful and complementary tools.

Wait or help for more nice features in pgBadger !

Cheers !

PS: There's another tool to deal with log files and reports you might be
interested in: "logwatch"

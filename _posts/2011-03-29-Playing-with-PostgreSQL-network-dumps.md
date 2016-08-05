---
layout: post
author: ioguix
title: Let's play with PostgreSQL network dumps!
date: 2011-03-29 11:30 +0200
tags: [pgshark, postgresql, network]
category: postgresql
---
A bit more than a year ago, I start messing with tshark
[pdml output](http://www.nbee.org/doku.php?id=netpdl:pdml_specification) to be
able to extract queries from a network dump.  As I was writing a PoC, I did it
in PHP, before moving it quickly to a more appropriate language: Perl.

{{ post.excerpt }}

As it was relying on tshark, I called it « pgShark ».  pgShark worked and was
useful a couple of time, but had some drawbacks:

* really, really slow
* PHP is not the best language to parse and mess with data
* one useless step: pcap -> XML/PDML -> parsing
* I have no fun coding in PHP
* support only a limited part of the PostgreSQL protocol

So I decided to rewrite it from scratch using Perl and the `Net::Pcap` module.

Why Perl?  First, probably the most frequent reasons people are using Perl are:
portable and fast about parsing data.  Then, I discovered the useful Net::Pcap
module, directly binded to the libpcap.  And finally, I wanted to learn a bit
more about Perl than just quick scripting and I needed a challenging project :)

At first I wanted a very simple program, dealing with frontend messages only
and using a plugin architecture to process them and output useful and various
informations.  But then, I realized it could go way further, crazy ideas about
plugins popped up and I realized that I had to support the whole PostgreSQL's
protocol, not just a small part of it.

The project started in early February 2011 and eat all my open source related
personal and professional time until now.  I finally managed to add the last
missing PostgreSQL messages on Monday 28th march.  I guess the pgShark::Core
module is now stable enough to focus on plugins.  I'll have to split this
project in two pieces at some point and release this core on its own.

One of the most exciting plugin is "Fouine".  Yeah, if you know pgFouine, you
understand this plugin's purpose (and if you don't,
[here is the pgfouine project](http://pgfouine.projects.postgresql.org/)).
Having Fouine as a plugin of pgShark has many advantages.  One of the most
important one is the ability to extract way much more statistics from network.
Think about errors, notices, session time, busy ratio, amount of data per
sessions, connections statistics, you name it...

Another really important point is that you don't need to tweak your PostgreSQL
log behaviour.  No need to log every query, no need to change the
`log_line_header` parameter or using syslog if you don't want to.  You just
leave your PostgreSQL configuration the way you love it.  Even better: working
on network dumps means you can snif anywhere in between the frontend and the
backend, having 0% performance penalty on your PostgreSQL box.

Check the existing plugin from the pgshark repository:
[github.com/dalibo/pgshark/tree/master/bin]()

The plugins are all in alpha development stage and need some more work for
accuracy or bug fix, but they already spit some useful datas!

Here is my TODO list for plugins:

* fix SQL plugin in regards with named portals (prepared statements with binded
  datas) 
* work on the [TODO](https://github.com/dalibo/pgshark/raw/34ae7706611778f8f012c9980f7d714def78b6e7/pgShark/Fouine.pm)
  list for the Fouine plugin (includes accuracy of statistics)
* make graphical reports (HTML) from the Fouine plugin
* graph various stats from the Fouine plugin

I have another really useful plugin idea, but really challenging.  Hopefully
I'll be able to start it and blog about it soon!  However, I suspect I'll have
to take some of my time on phpPgAdmin very soon, as I was pretty much idle on
it for a while now and stuff are staking on this TODO list as well.

Don't forget the pgShark's home: [github.com/dalibo/pgshark](https://github.com/dalibo/pgshark).
Stay tuned!

---
layout: post
author: ioguix
title: Build a PostreSQL Automated Failover in 5 minutes
date: 2019-01-24 23:00:00 +0200
tags: [postgresql, ha, pacemaker]
category: postgresql
cast: true
---

I've been working with [Vagrant](https://www.vagrantup.com/) to quickly build a fresh
test cluster for [PAF](https://clusterlabs.github.io/PAF/) development. Combined with
`virsh` snapshot related commands, I save a lot of time when quickly rollback the whole
cluster to a stable situation.

I thought the result might be useful for people that want to challenge such a
cluster, without all the burden to building it and setting everything it up. Vagrantfile
and related files has been pushed to the official PAF repository, just
look at the `test/README.md` file there for complete instructions:
[https://github.com/ClusterLabs/PAF](https://github.com/ClusterLabs/PAF).

After very short and easy prerequisites, you'll be able to build a fresh and ready
cluster with one command and a cup of coffee. It takes about 4 minutes on my laptop to
complete, from creating four new empty VMs to watching at the full cluster status.

And for the busiest of you that want to see it in action, here is a screencast (about 4
minutes, 7'30" in real speed). It starts on a virgin fedora 29, all prerequisites and
setup are covered, from scratch to a running virtualized demo cluster:

{% include screencast-building-paf.html %}

And if you want some examples about how to exercise the cluster look at this
other screencast (5'32, normal speed):

{% include screencast-testing-paf.html %}

Note that all log from cluster nodes are collected on `log-sink` in `/var/log/messages` and
`/var/log/<nodename>`.

If you want to go deeper with PAF, do not forget to read our documentation with
attention. HA is not a matter of some commands copy/pasted from internet. It requires
some time, documentation and lot of tests. HINT: write docs and procedures, lots of them,
and keep them up to date!

Cheers,

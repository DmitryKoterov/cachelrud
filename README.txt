CacheLRUd: implements cache LRU cleanup on various databases (e.g. MongoDB)
Version: 0.50
Author: Dmitry Koterov, dkLab (C)
GitHub: http://github.com/DmitryKoterov/
License: GPL2


MOTIVATION
----------

Sometimes (not always, but in many projects) it's quite handy to use some
database (e.g. MongoDB) instead of plain-old memcached. This is probably
your case if your cache is:

a) relatively small (hundreds of GBs in one replica set, no more);
b) contains many long-living keys (live for days);
c) strongly tagged, and tags cleanup robustness is really important;
d) needs to be replicated through multiple machines or datacenters;
e) possibly is sharded (where MongoDB is very good at).

So, probably in these cases it would be good to use MongoDB (or another
database) as a caching engine. MongoDB is very fast (almost as fast as
memcached on reads), it supports replication with auto-failover, could
be sharded etc. But it does not implement an LRU cleaning algorithm, and
you cannot, of course, update a "last hit" field in your collection on
each cache read hit synchronously (especially if MongoDB master is in
another datacenter). So there is no easy way to keep the database size
constant.

CacheLRUd tries to solve this problem: it looks after your database size
and removes keys which were not READ for too long.

But how CacheLRUd knows which keys were READ recently? Your caching layer
should notify the daemon on which keys were read recently by sending UDP
packets to it. UDP is asynchronous and does not block your application, so
you may even send an UDP packet per each cache hit pack, even to another
datacenter (if you do not have millions of page hits per second, of course).
To eliminate single point of failure, install CacheLRUd service on each
MongoDB node and send UDP notifications to alive nodes only.


HOW TO SEND UDP PACKETS
-----------------------

To notify CacheLRUd that a cache key "key" has been read recently in a
collection configured as "[collection_name]" in /etc/cachelrud.conf,
just send an UDP packet to the daemon's port (defaults to 43521):

    collection_name:key

If you have many hits, you may group them and send in a single UDP
message separated by newline characters (to save bandwidth):

    collection_name:key1
    collection_name:key2
    ...


INSTALLATION ON LINUX
---------------------

## Install the service on EACH MongoDB NODE:
cd /opt
git clone git@github.com:DmitryKoterov/cachelrud.git
ln -s /opt/cachelrud/cachelrud.init /etc/init.d/cachelrud

## Configure:
cp /opt/cachelrud/cachelrud.conf /etc/cachelrud.conf  # and then edit

## For RHEL (RedHat, CentOS):
chkconfig --add cachelrud
chkconfig cachelrud on

## ...or for Debian/Ubuntu:
update-rc.d cachelrud defaults
update-rc.d cachelrud start


SUPPORT FOR YOUR FAVORITE DATABASE
----------------------------------

CacheLRUd is written in Python. To make it support a new database,
you may create a file lib/cachelrud/storage/your_database_name.py
(use mongodb.py for some inspiration).
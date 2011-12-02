Erlang Block Device
===================

What is it
----------

A distributed block device written in Erlang (userspace) and C (kernel space).

If you need *fast synchronous writes* (for example, running a relational database), you have three choices:

1. Battery-backed RAID cards. This works, but is limited by RAID controller internal memory and quite unsafe in practice.
1. SSD disks. This works well, but SSDs are still too slow.
1. Make use of SAS Multipath feature.

ErlBD implements third.

How it works
------------
            +----------+
            |  Client  |
            +----------+
            |          |
            | Ethernet |
            |          |
    +-------+--+    +--+-------+
    |   SAN1   |    |   SAN2   |
    +----+-----+    +-----+----+
         |                |
     SAS +-----+    +-----+ SAS
               |    |
            +--+----+--+
            | SAS disk |
            +----------+

Client is the ErlBD client. SAN1 and SAN2 are ErlBD servers. SAN1 is "master",
SAN2 is "slave". Both SANs are connected to the SAS disk via multipath I/O.

When a client does a write(), data is sent to both SAN1 and SAN2. When SAN2
receives a data blob, it sends "ACK" to SAN1. **When SAN1 gets blob and the
message from SAN2, SAN1 can safely treat data as fsync'd**.

We have a safe fsync, while data is still in RAM!

When SAN1 dies, SAN2 writes all unwritten data to disk and takes the "master" role. When there is only one node that does the writes, fsyncs are obviously handled in a traditional way.

Why Erlang?
-----------

1. Rock-solid message passing for free
1. Erlang failure and error handling abilities
1. [gen_leader](http://groups.google.com/group/erlang-programming/browse_thread/thread/5f502cccff0b1fa2/f1a28a5c9f69bcad?#f1a28a5c9f69bcad)

This code is not supposed to be on production due to obvious performance reasons.

My hypothesis
-------------

Middle-to-large writes (100K * 10K, 10K * 100K, 1K * 1M) will be faster with ErlBD compared to a decent SSD. Precise testing conditions are to be done.

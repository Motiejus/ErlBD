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

Client is the ErlBD client. SAN1 and SAN2 are ErlBD servers. SAN1 is "Master",
SAN2 is "Slave". Both SANs are connected to the SAS disk via multipath I/O.

When a client does a `write()`, data is sent to both SAN1 and SAN2. When SAN2
receives a data blob, it sends `ACK` to SAN1. **When SAN1 gets blob and the
message from SAN2, SAN1 can safely treat data as `fsync`'d**.

We have a safe fsync, while data is still in RAM!

When SAN1 dies, SAN2 writes all unwritten data to disk and takes the "Master" role. When there is only one node that does the writes, fsyncs are handled in a traditional way.

Why Erlang?
-----------

1. Rock-solid message passing for free
1. Erlang failure and error handling abilities
1. [gen_leader](http://groups.google.com/group/erlang-programming/browse_thread/thread/5f502cccff0b1fa2/f1a28a5c9f69bcad?#f1a28a5c9f69bcad)

My hypothesis
-------------

Middle-to-large writes (100K * 10K, 10K * 100K, 1K * 1M) will be faster with ErlBD compared to a decent SSD. Precise testing conditions are to be done once something working is implemented.

More details
------------

So, high level overview is not sufficient for you? Read on.

Client:

* Kernel block device driver `erlbd` (`erlbd.c`)
* Erlang node (client)

SAN1: `fd1e:3ac8:0511:2250::1`
SAN2: `fd1e:3ac8:0511:2250::2`
Client: `fd1e:3ac8:0511:2250::200`

SAN1 (Master) Initialization:

    $ erlbd-server --client=fd1e:3ac8:0511:2250::200 --slave=fd1e:3ac8:0511:2250::2 /dev/mapper/mpath0

SAN2 (Slave) Initialization:

    $ erlbd-server --client=fd1e:3ac8:0511:2250::200 --master=fd1e:3ac8:0511:2250::2 /dev/mapper/mpath0

Client initialization:

    $ erlbd-client --server=fd1e:3ac8:0511:2250::1 --server=fd1e:3ac8:0511:2250::2
    $ mount /dev/erlbd0 /mnt/yadda

What happens in the client
--------------------------

1. Kernel driver loads, initializes `/sys/block/erlbd{0-15}/`
1. Erlang client node is started:
    - Connect to server node and get `{Block_size, Number_of_sectors}`
    - Pass those values to kernel driver via sysfs interface
    - Kernel driver registers the block device in the kernel (`register_blkdev()`)
1. When client does a `write()` and kernel flushes:
    - `erlbd_request()` function in kernel driver is called with the data blob
    - which sends data to the local Erlang node using [Erlang Port interface](http://www.erlang.org/doc/tutorial/c_portdriver.html)
    - which in turn sends `{Ref :: reference(), Blob :: binary()}` to both Server nodes
    - Erlang client node receives `{flushed, Ref}` from Server node
    - sends `flushed` to the kernel driver
    - driver driver completes the `erlbd_request()`

What happens in the server nodes
--------------------------------

1. A server node gets a Request = `{Ref, Blob}` from the client:
    - Slave node:
        * saves the `Blob` to RAM (state variable is most likely)
        * sends `{ack, Ref}` to Master node
    - Master node:
        * queues I/O operation to disk
        * after any one of these events:
            1. I/O operation completes
            1. `{ack, Ref}` received from Slave node

            Sends `{flushed, Ref}` to the client
        * when I/O operation completes:
            - sends `{completed, Ref}` to the Slave node, which, when receives this term, drops associated `Blob` from memory
1. When Slave node crashes, nothing changes in the scheme.
1. When Master node crashes, Slave node:
    - queues all pending I/O operations to disk
    - sets its status to Master and starts behaving like Master
1. When previous Master node comes back:
    - Registers itself to current Master as Slave
    - Operation continues ...

What about changing statuses from Master to Slave? Should be trivial to do.

Redis cluster tutorial
===

This document is a gentle introduction to Redis Cluster, that does not use
complex to understand distributed systems concepts. It provides instructions
about how to setup a cluster, test, and operate it, without
going into the details that are covered in
the [Redis Cluster specification](/topics/cluster-spec) but just describing
how the system behaves from the point of view of the user.

Note that if you plan to run a serious Redis Cluster deployment, the
more formal specification is an highly suggested reading.

**Redis cluster is currently alpha quality code**, please get in touch in the
Redis mailing list or open an issue in the Redis Github repository if you
find any issue.

Redis Cluster 101
---

Redis Cluster provides a way to run a Redis installation where data is
**automatically sharded across multiple Redis nodes**.

Commands dealing with multiple keys are not supported by the cluster, because
this would require moving data between Redis nodes, making Redis Cluster
not able to provide Redis-alike performances and predictable behavior
under load.

Redis Cluster also provides **some degree of availability during partitions**,
that is in practical terms the ability to continue the operations when
some nodes fail or are not able to communicate.

So in practical terms, what you get with Redis Cluster?

* The ability to automatically split your dataset among multiple nodes.
* The ability to continue operations when a subset of the nodes are experiencing failures or are unable to communicate with the rest of the cluster.

Redis Cluster data sharding
---

Redis Cluster does not use consistency hashing, but a different form of sharding
where every key is conceptually part of what we call an **hash slot**.

There are 16384 hash slots in Redis Cluster, and to compute what is the hash
slot of a given key, we simply take the CRC16 of the key modulo
16384.

Every node in a Redis Cluster is responsible of a subset of the hash slots,
so for example you may have a cluster with 3 nodes, where:

* Node A contains hash slots from 0 to 5500.
* Node B contains hash slots from 5501 to 11000.
* Node C contains hash slots from 11001 to 16384.

This allows to add and remove nodes in the cluster easily. For example if
I want to add a new node D, I need to move some hash slot from nodes A, B, C
to D. Similarly if I want to remove node A from the cluster I can just
move the hash slots served by A to B and C. When the node A will be empty
I can remove it from the cluster completely.

Because moving hash slots from a node to another does not require to stop
operations, adding and removing nodes, or changing the percentage of hash
slots hold by nodes, does not require any downtime.

Redis Cluster master-slave model
---

In order to remain available when a subset of nodes are failing or are not able
to communicate with the majority of nodes, Redis Cluster uses a master-slave
model where every node has from 1 (the master itself) to N replicas (N-1
additional slaves).

In our example cluster with nodes A, B, C, if node B fails the cluster is not
able to continue, since we no longer have a way to serve hash slots in the
range 5501-11000.

However if when the cluster is created (or at a latter time) we add a slave
node to every master, so that the final cluster is composed of A, B, C
that are masters, and A1, B1, C1 that are slaves, the system is able to
continue if node B fails.

Node B1 replicates B, so the cluster will elect node B1 as the new master
and will continue to operate correctly.

However note that if nodes B and B1 fail at the same time Redis Cluster is not
able to continue to operate.

Redis Cluster consistency guarantees
---

Redis Cluster is not able to guarantee **strong consistency**. In practical
terms this means that under certain conditions it is possible that Redis
Cluster will forget a write that was acknowledged by the system.

The first reason why Redis Cluster can lose writes is because it uses
asynchronous replication. This means that during writes the following
happens:

* Your client writes to the master B.
* The master B replies OK to your client.
* The master B propagates the write to its slaves B1, B2 and B3.

As you can see B does not wait for an acknowledge from B1, B2, B3 before
replying to the client, since this would be a prohibitive latency penalty
for Redis, so if your client writes something, B acknowledges the write,
but crashes before being able to send the write to its slaves, one of the
slaves can be promoted to master losing the write forever.

This is **very similar to what happens** with most databases that are
configured to flush data to disk every second, so it is a scenario you
are already able to reason about because of past experiences with traditional
database systems not involving distributed systems. Similarly you can
improve consistency by forcing the database to flush data on disk before
replying to the client, but this usually results into prohibitively low
performances.

Basically there is a trade-off to take between performances and consistency.

Note: Redis Cluster in the future will allow users to perform synchronous
writes when absolutely needed.

There is another scenario where Redis Cluster will lose writes, that happens
during a network partition where a client is isolated with a minority of
instances including at least a master.

Take as an example our 6 nodes cluster composed of A, B, C, A1, B1, C1,
with 3 masters and 3 slaves. There is also a client, that we will call Z1.

After a partition occurs, it is possible that in one side of the
partition we have A, C, A1, B1, C1, and in the other side we have B and Z1.

Z1 is still able to write to B, that will accept its writes. If the
partition heals in a very short time, the cluster will continue normally.
However if the partition lasts enough time for B1 to be promoted to master
in the majority side of the partition, the writes that Z1 is sending to B
will be lost.

Note that there is a maximum window to the amount of writes Z1 will be able
to send to B: if enough time has elapsed for the majority side of the
partition to elect a slave as master, every master node in the minority
side stops accepting writes.

This amount of time is a very important configuration directive of Redis
Cluster, and is called the **node timeout**.

After node timeout has elapsed, a master node is considered to be failing,
and can be replaced by one if its replicas.
Similarly after node timeout has elapsed without a master node to be able
to sense the majority of the other master nodes, it enters an error state
and stops accepting writes.

Creating and using a Redis Cluster
===

To create a cluster, the first thing we need is to have a few empty
Redis instances running in **cluster mode**. This basically means that
clusters are not created using normal Redis instances, but a special mode
needs to be configured so that the Redis instance will enable the Cluster
specific features and commands.

The following is a minimal Redis cluster configuration file:

```
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

As you can see what enables the cluster mode is simply the `cluster-enabled`
directive. Every instance also contains the path of a file where the
configuration for this node is stored, that by default is `nodes.conf`.
This file is never touched by humans, it is simply generated at startup
by the Redis Cluster instances, and updated every time it is needed.

Note that the **minimal cluster** that works as expected requires to contain
at least three master nodes. For your first tests it is strongly suggested
to start a six nodes cluster with three masters and three slaves.

To do so, enter a new directory, and create the following directories named
after the port number of the instance we'll run inside any given directory.

Something like:

```
mkdir cluster-test
cd cluster-test
mkdir 7000 7001 7002 7003 7004 7005
```

Create a `redis.conf` file inside each of the directories, from 7000 to 7005.
As a template for your configuration file just use the small example above,
but make sure to replace the port number `7000` with the right port number
according to the directory name.

Now copy your redis-server executable, **compiled from the latest sources in the unstable branch at Github**, into the `cluster-test` directory, and finally open 6 terminal tabs in your favorite terminal application.

Start every instance like that, one every tab:

```
cd 7000
../redis-server ./redis.conf
```

As you can see from the logs of every instance, since no `nodes.conf` file
existed, every node assigns itself a new ID.

    [82462] 26 Nov 11:56:55.329 * No cluster configuration found, I'm 97a3a64667477371c4479320d683e4c8db5858b1

This ID will be used forever by this specific instance in order for the instance
to have an unique name in the context of the cluster. Every node
remembers every other node using this IDs, and not by IP or port.
IP addresses and ports may change, but the unique node identifier will never
change for all the life of the node. We call this identifier simply **Node ID**.

Creating the cluster
---

Now that we have a number of instances running, we need to create our
cluster writing some meaningful configuration to the nodes.

This is very easy to accomplish as we are helped by the Redis Cluster
command line utility called `redis-trib`, that is a Ruby program
executing special commands in the instances in order to create new clusters,
check or reshard an existing cluster, and so forth.

The `redis-trib` utility is in the `src` directory of the Redis source code
distribution. To create your cluster simply type:

    ./redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 \
    127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005

The command used here is **create**, since we want to create a new cluster.
The option `--replicas 1` means that we want a slave for every master created.
The other arguments are the list of addresses of the instances I want to use
to create the new cluster.

Obviously the only setup with our requirements is to create a cluster with
3 masters and 3 slaves.

Redis-trib will propose you a configuration. Accept typing **yes**.
The cluster will be configured and *joined*, that means, instances will be
bootstrapped into talking with each other. Finally if everything went ok
you'll see a message like that:

    [OK] All 16384 slots covered

This means that there is at least a master instance serving each of the
16384 slots available.

Playing with the cluster
---

At this stage one of the problems with Redis Cluster is the lack of
client libraries implementations.

I'm aware of the following implementations:

* [redis-rb-cluster](http://github.com/antirez/redis-rb-cluster) is a Ruby implementation written by me (@antirez) as a reference for other languages. It is a simple wrapper around the original redis-rb, implementing the minimal semantics to talk with the cluster efficiently.
* [redis-py-cluster](https://github.com/Grokzen/redis-py-cluster) appears to be a port of redis-rb-cluster to Python. Not recently updated (last commit 6 months ago) however it may be a starting point.
* The popular [Predis](https://github.com/nrk/predis) has support for Redis Cluster, the support was recently updated and is in active development.
* The most used Java client, [Jedis](https://github.com/xetorthio/jedis) recently added support for Redis Cluster, see the *Jedis Cluster* section in the project README.
* The `redis-cli` utility in the unstable branch of the Redis repository at Github implements a very basic cluster support when started with the `-c` switch.

An easy way to test Redis Cluster is either to try and of the above clients
or simply the `redis-cli` command line utility. The following is an example
of interaction using the latter:

```
$ redis-cli -c -p 7000
redis 127.0.0.1:7000> set foo bar
-> Redirected to slot [12182] located at 127.0.0.1:7002
OK
redis 127.0.0.1:7002> set hello world
-> Redirected to slot [866] located at 127.0.0.1:7000
OK
redis 127.0.0.1:7000> get foo
-> Redirected to slot [12182] located at 127.0.0.1:7002
"bar"
redis 127.0.0.1:7000> get hello
-> Redirected to slot [866] located at 127.0.0.1:7000
"world"
```

The redis-cli cluster support is very basic so it always uses the fact that
Redis Cluster nodes are able to redirect a client to the right node.
A serious client is able to do better than that, and cache the map between
hash slots and nodes addresses, to directly use the right connection to the
right node. The map is refreshed only when something changed in the cluster
configuration, for example after a failover or after the system administrator
changed the cluster layout by adding or removing nodes.

Writing an example app with redis-rb-cluster
---

Before goign forward showing how to operate the Redis Cluster, doing things
like a failover, or a resharding, we need to create some example application
or at least to be able to understand the semantics of a simple Redis Cluster
client interaction.

In this way we can run an example and at the same time try to make nodes
failing, or start a resharding, to see how Redis Cluster behaves under real
world conditions. It is not very helpful to see what happens while nobody
is writing to the cluster.

This section explains some basic usage of redis-rb-cluster showing two
examples. The first is the following, and is the `example.rb` file inside
the redis-rb-cluster distribution:

```
     1  require './cluster'
     2
     3  startup_nodes = [
     4      {:host => "127.0.0.1", :port => 7000},
     5      {:host => "127.0.0.1", :port => 7001}
     6  ]
     7  rc = RedisCluster.new(startup_nodes,32,:timeout => 0.1)
     8
     9  last = false
    10
    11  while not last
    12      begin
    13          last = rc.get("__last__")
    14          last = 0 if !last
    15      rescue => e
    16          puts "error #{e.to_s}"
    17          sleep 1
    18      end
    19  end
    20
    21  ((last.to_i+1)..1000000000).each{|x|
    22      begin
    23          rc.set("foo#{x}",x)
    24          puts rc.get("foo#{x}")
    25          rc.set("__last__",x)
    26      rescue => e
    27          puts "error #{e.to_s}"
    28      end
    29      sleep 0.1
    30  }
```

The application does a very simple thing, it sets keys in the form `foo<number>` to `number`, one after the other. So if you run the program the result is the
following stream of commands:

* SET foo0 0
* SET foo1 1
* SET foo2 2
* And so forth...

The program looks more complex than it should usually as it is designed to
show errors on the screen instead of exiting with an exception, so every
operation performed with the cluster is wrapped by `begin` `rescue` blocks.

The **line 7** is the first interesting line in the program. It creates the
Redis Cluster object, using as argument a list of *startup nodes*, the maximum
number of connections this object is allowed to take against different nodes,
and finally the timeout after a given operation is considered to be failed.

The startup nodes don't need to be all the nodes of the cluster. The important
thing is that at least one node is reachable. Also note that redis-rb-cluster
updates this list of startup nodes as soon as it is able to connect with the
first node. You should expect such a behavior with any other serious client.

Now that we have the Redis Cluster object instance stored in the **rc** variable
we are ready to use the object like if it was a normal Redis object instance.

This is exactly what happens in **line 11 to 19**: when we restart the example
we don't want to start again with `foo0`, so we store the counter inside
Redis itself. The code above is designed to read this counter, or if the
counter does not exist, to assign it the value of zero.

However note how it is a while loop, as we want to try again and again even
if the cluster is down and is returning errors. Normal applications don't need
to be so careful.

**Lines between 21 and 30** start the main loop where the keys are set or
an error is displayed.

Note the `sleep` call at the end of the loop. In your tests you can remove
the sleep if you want to write to the cluster as fast as possible (relatively
to the fact that this is a busy loop without real parallelism of course, so
you'll get the usually 10k ops/second in the best of the conditions).

Normally writes are slowed down in order for the example application to be
easier to follow by humans.

Starting the application produces the following output:

```
ruby ./example.rb
1
2
3
4
5
6
7
8
9
^C (I stopped the program here)
```

This is not a very interesting program and we'll use a better one in a moment
but we can already try what happens during a resharding when the program
is running.

Resharding the cluster
---

Now we are ready to try a cluster resharding. To do this please
keep the example.rb program running, so that you can see if there is some
impact on the program running. Also you may want to comment the `sleep`
call in order to have some more serious write load during resharding.

Resharding basically means to move hash slots from a set of nodes to another
set of nodes, and like cluster creation it is accomplished using the
redis-trib utility.

To start a resharding just type:

    ./redis-trib.rb reshard 127.0.0.1:7000

You only need to specify a single node, redis-trib will find the other nodes
automatically.

Currently redis-trib is only able to reshard with the administrator support,
you can't just say move 5% of slots from this node to the other one (but
this is pretty trivial to implement). So it starts with questions. The first
is how much a big resharding do you want to do:

    How many slots do you want to move (from 1 to 16384)?

We can try to reshard 1000 hash slots, that should already contain a non
trivial amount of keys if the example is still running without the sleep
call.

Then redis-trib needs to know what is the target of the resharding, that is,
the node that will receive the hash slots.
I'll use the first master node, that is, 127.0.0.1:7000, but I need
to specify the Node ID of the instance. This was already printed in a
list by redis-trib, but I can always find the ID of a node with the following
command if I need:

```
$ redis-cli -p 7000 cluster nodes | grep myself
97a3a64667477371c4479320d683e4c8db5858b1 :0 myself,master - 0 0 0 connected 0-5460
```

Ok so my target node is 97a3a64667477371c4479320d683e4c8db5858b1.

Now you'll get asked from what nodes you want to take those keys.
I'll just type `all` in order to take a bit of hash slots from all the
other master nodes.

After the final confirmation you'll see a message for every slot that
redis-trib is going to move from a node to another, and a dot will be printed
for every actual key moved from one side to the other.

While the resharding is in progress you should be able to see your
example program running unaffected. You can stop and restart it multiple times
during the resharding if you want.

At the end of the resharding, you can test the health of the cluster with
the following command:

    ./redis-trib.rb check 127.0.0.1:7000

All the slots will be covered as usually, but this time the master at
127.0.0.1:7000 will have more hash slots, something around 6461.

A more interesting example application
---

So far so good, but the example application we used is not very good.
It writes acritically to the cluster without ever checking if what was
written is the right thing.

From our point of view the cluster receiving the writes could just always
write the key `foo` to `42` to every operation, and we would not notice at
all.

So in the reids-rb-cluster repository, there is a more interesting application
that is called `consistency-test.rb`. It is a much more interesting application
as it uses a set of counters, by default 1000, and sends `INCR` commands
in order to increment the counters.

However instead of just writing, the application does two additional things:

* When a counter is updated using `INCR`, the application remembers the write.
* It also reads a random counter before every write, and check if the value is what it expected it to be, comparing it with the value it has in memory.

What this means is that this application is a simple **consistency checker**,
and is able to tell you if the cluster lost some write, or if it accepted
a write that we did not received acknowledgement for. In the first case we'll
see a counter having a value that is smaller than the one we remember, while
in the second case the value will be greater.

Running the consistency-test application produces a line of output every
second:

```
$ ruby consistency-test.rb
925 R (0 err) | 925 W (0 err) |
5030 R (0 err) | 5030 W (0 err) |
9261 R (0 err) | 9261 W (0 err) |
13517 R (0 err) | 13517 W (0 err) |
17780 R (0 err) | 17780 W (0 err) |
22025 R (0 err) | 22025 W (0 err) |
25818 R (0 err) | 25818 W (0 err) |
```

The line shows the number of **R**eads and **W**rites performed, and the
number of errors (query not accepted because of errors since the system was
not available).

If some inconsistency is found, new lines are added to the output.
This is what happens, for example, if I reset a counter manually while
the program is running:

```
$ redis 127.0.0.1:7000> set key_217 0
OK

(in the other tab I see...)

94774 R (0 err) | 94774 W (0 err) |
98821 R (0 err) | 98821 W (0 err) |
102886 R (0 err) | 102886 W (0 err) | 114 lost |
107046 R (0 err) | 107046 W (0 err) | 114 lost |
```

When I set the counter to 0 the real value was 144, so the program reports
144 lost writes (`INCR` commands that are not remembered by the cluster).

This program is much more interesting as a test case, so we'll use it
to test the Redis Cluster failover.

Testing the failover
---

Note: during this test, you should take a tab open with the consistency test
application running.

In order to trigger the failover, the simplest thing we can do (that is also
the semantically simplest failure that can occur in a distributed system)
is to crash a single process, in our case a single master.

We can identify a cluster and crash it with the following command:

```
$ redis-cli -p 7000 cluster nodes | grep master
3e3a6cb0d9a9a87168e266b0a0b24026c0aae3f0 127.0.0.1:7001 master - 0 1385482984082 0 connected 5960-10921
2938205e12de373867bf38f1ca29d31d0ddb3e46 127.0.0.1:7002 master - 0 1385482983582 0 connected 11423-16383
97a3a64667477371c4479320d683e4c8db5858b1 :0 myself,master - 0 0 0 connected 0-5959 10922-11422
```

Ok, so 7000, 7001, and 7002 are masters. Let's crash node 7002 with the
**DEBUG SEGFAULT** command:

```
$ redis-cli -p 7002 debug segfault
Error: Server closed the connection
```

Now we can look at the output of the consistency test to see what it reported.

```
18849 R (0 err) | 18849 W (0 err) |
23151 R (0 err) | 23151 W (0 err) |
27302 R (0 err) | 27302 W (0 err) |

... many error warnings here ...

29659 R (578 err) | 29660 W (577 err) |
33749 R (578 err) | 33750 W (577 err) |
37918 R (578 err) | 37919 W (577 err) |
42077 R (578 err) | 42078 W (577 err) |
```

As you can see during the failover the system was not able to accept 578 reads and 577 writes, however no inconsistency was created in the database. This may
sound unexpected as in the first part of this tutorial we stated that Redis
Cluster can lost writes during the failover because it uses asynchronous
replication. What we did not said is that this is not very likely to happen
because Redis sends the reply to the client, and the commands to replicate
to the slaves, about at the same time, so there is a very small window to
lose data. However the fact that it is hard to trigger does not mean that it
is impossible, so this does not change the consistency guarantees provided
by Redis cluster.

We can now check what is the cluster setup after the failover (note that
in the meantime I restarted the crashed instance so that it rejoins the
cluster as a slave):

```
$ redis-cli -p 7000 cluster nodes
3fc783611028b1707fd65345e763befb36454d73 127.0.0.1:7004 slave 3e3a6cb0d9a9a87168e266b0a0b24026c0aae3f0 0 1385503418521 0 connected
a211e242fc6b22a9427fed61285e85892fa04e08 127.0.0.1:7003 slave 97a3a64667477371c4479320d683e4c8db5858b1 0 1385503419023 0 connected
97a3a64667477371c4479320d683e4c8db5858b1 :0 myself,master - 0 0 0 connected 0-5959 10922-11422
3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 127.0.0.1:7005 master - 0 1385503419023 3 connected 11423-16383
3e3a6cb0d9a9a87168e266b0a0b24026c0aae3f0 127.0.0.1:7001 master - 0 1385503417005 0 connected 5960-10921
2938205e12de373867bf38f1ca29d31d0ddb3e46 127.0.0.1:7002 slave 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 0 1385503418016 3 connected
```

Now the masters are running on ports 7000, 7001 and 7005. What was previously
a master, that is the Redis instance running on port 7002, is now a slave of
7005.

The output of the `CLUSTER NODES` command may look intimidating, but it is actually pretty simple, and is composed of the following tokens:

* Node ID
* ip:port
* flags: master, slave, myself, fail, ...
* if it is a slave, the Node ID of the master
* Time of the last pending PING still waiting for a reply.
* Time of the last PONG received.
* Configuration epoch for this node (see the Cluster specification).
* Status of the link to this node.
* Slots served...

Manual failover
---

Sometimes it is useful to force a failover without actually causing any problem
on a master. For example in order to upgrade the Redis process of one of the
master nodes it is a good idea to failover it in order to turn it into a slave
with minimal impact on availability.

Manual failovers are supported by Redis Cluster using the `CLUSTER FAILOVER`
command, that must be executed in one of the **slaves** of the master you want
to failover.

Manual failovers are special and are safer compared to failovers resulting from
actual master failures, since they occur in a way that avoid data loss in the
process, by switching clients from the original master to the new master only
when the system is sure that the new master processed all the replication stream
from the old one.

This is what you see in the slave log when you perform a manual failover:

    # Manual failover user request accepted.
    # Received replication offset for paused master manual failover: 347540
    # All master replication stream processed, manual failover can start.
    # Start of election delayed for 0 milliseconds (rank #0, offset 347540).
    # Starting a failover election for epoch 7545.
    # Failover election won: I'm the new master.

Basically clients connected to the master we are failing over are stopped.
At the same time the master sends its replication offset to the slave, that
waits to reach the offset on its side. When the replication offset is reached,
the failover starts, and the old master is informed about the configuration
switch. When the clients are unblocked on the old master, they are redirected
to the new master.

Adding a new node
---

Adding a new node is basically the process of adding an empty node and then
moving some data into it, in case it is a new master, or telling it to
setup as a replica of a known node, in case it is a slave.

We'll show both, starting with the addition of a new master instance.

In both cases the first step to perform is **adding an empty node**.

This is as simple as to start a new node in port 7006 (we already used
from 7000 to 7005 for our existing 6 nodes) with the same configuration
used for the other nodes, except for the port number, so what you should
do in order to conform with the setup we used for the previous nodes:

* Create a new tab in your terminal application.
* Enter the `cluster-test` directory.
* Create a directory named `7006`.
* Create a redis.conf file inside, similar to the one used for the other nodes but using 7006 as port number.
* Finally start the server with `../redis-server ./redis.conf`

At this point the server should be running.

Now we can use **redis-trib** as usually in order to add the node to
the existing cluster.

    ./redis-trib.rb addnode 127.0.0.1:7006 127.0.0.1:7000

As you can see I used the **addnode** command specifying the address of the
new node as first argument, and the address of a random existing node in the
cluster as second argument.

In practical terms redis-trib here did very little to help us, it just
sent a `CLUSTER MEET` message to the node, something that is also possible
to accomplish manually. However redis-trib also checks the state of the
cluster before to operate, that is an advantage, and will be improved more
and more in the future in order to also be able to rollback changes when
needed or to help the user to fix a messed up cluster when there are issues.

Now we can connect to the new node to see if it really joined the cluster:

```
redis 127.0.0.1:7006> cluster nodes
3e3a6cb0d9a9a87168e266b0a0b24026c0aae3f0 127.0.0.1:7001 master - 0 1385543178575 0 connected 5960-10921
3fc783611028b1707fd65345e763befb36454d73 127.0.0.1:7004 slave 3e3a6cb0d9a9a87168e266b0a0b24026c0aae3f0 0 1385543179583 0 connected
f093c80dde814da99c5cf72a7dd01590792b783b :0 myself,master - 0 0 0 connected
2938205e12de373867bf38f1ca29d31d0ddb3e46 127.0.0.1:7002 slave 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 0 1385543178072 3 connected
a211e242fc6b22a9427fed61285e85892fa04e08 127.0.0.1:7003 slave 97a3a64667477371c4479320d683e4c8db5858b1 0 1385543178575 0 connected
97a3a64667477371c4479320d683e4c8db5858b1 127.0.0.1:7000 master - 0 1385543179080 0 connected 0-5959 10922-11422
3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 127.0.0.1:7005 master - 0 1385543177568 3 connected 11423-16383
```

Note that since this node is already connected to the cluster it is already
able to redirect client queries correctly and is generally speaking part of
the cluster. However it has two peculiarities compared to the other masters:

* It holds no data as it has no assigned hash slots.
* Because it is a master without assigned slots, it does not participate in the election process when a slave wants to become a master.

Now it is possible to assign hash slots to this node using the resharding
feature of `redis-trib`. It is basically useless to show this as we already
did in a previous section, so instead I want to cover the case where we want
to turn this instance into a replica (a slave) of some other master.

For example in order to add a replica for the node 127.0.0.1:7005 that is
currently serving hash slots in the range 11423-16383, that has a Node ID
3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e, all I need to do is to connect
with the new node that we just added and send a simple command:

    redis 127.0.0.1:7006> cluster replicate 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e

That's it. Now we have a new replica for this set of hash slots, and all
the other nodes in the cluster already know (after a few seconds needed to
update their config). We can verify with the following command:

```
$ redis-cli -p 7000 cluster nodes | grep slave | grep 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e
f093c80dde814da99c5cf72a7dd01590792b783b 127.0.0.1:7006 slave 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 0 1385543617702 3 connected
2938205e12de373867bf38f1ca29d31d0ddb3e46 127.0.0.1:7002 slave 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 0 1385543617198 3 connected
```

The node 3c3a0c... now has two slaves, running on ports 7002 (the existing one) and 7006 (the new one).

Removing a node
---

Work in progress.
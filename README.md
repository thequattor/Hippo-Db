HippoDB
========

HippoDB is a distributed database meant to serve data which lives in Hadoop clusters (but not only) to frontend applications. In order to keep things as simple and as robust as possible, HippoDB is a read-only key-value store. This means that:

* data is *never* written. Instead dumps for HippoDB are created externally and an atomic switch is made when a new version is available.
* the *only* pattern of access to resident data is **by key**, although there is support for splitting data relative to a single key into columns.

This restrictions allow to write a tool which has very few moving parts, and so it is extremely simple, both in terms of logic and in terms of lines of code (830 lines, excluding blanks and comments). Hence, although very young, HippoDB is also very robust.

It also allows to switch back to a past version, in the event that the last version deployed should contain erroneous data.

Workflow
--------

A typical workflow would have HippoDB work in tandem with another more sophisticated store such as **HBase**. Data comes in HBase and possibly batch jobs (for instance map-reduce) are run.

Periodically - say every two hours - relevant data is dumped to HippoDB for serving to frontend applications. Once it is ready, a command is issued to HippoDB, to instantaneously switch version. In case something went wrong, a rollback can be made with no downtime.

What HippoDB supports
----------------------

* Distributed stores with a chosen replication factor
* Multiget requests for any number of keys and columns
* Reliable serving even when a few servers fail
* Elastic growth of the cluster, with no downtime
* Native (JVM) or HTTP clients
* Clients perform automatic load-balancing between existing servers, in a cache-friendly way (so the same request will always be sent to the same server, if possible)

The third line needs a little explanation. If r is the replication factor, data can still be served if up to r-1 servers fail.

If more than this number fails, data that is present on the remaining servers will still be served without failures, although of course some data may only be present on the failed machines.

Finally, the client transparently switches to any server which is still live. This means that, although the client is initialized with the address of any of the servers, it will still be able to connect even if this particular server fails (although of the server needs to be up in the first instant when the client connects). In fact, requests are always load-balanced by clients between live servers.

Since HippoDB is read-only, the elastic growth of the cluster means that new servers will be recognized and will become part of the cluster with no downtime, but data will have to be dumped taking into account the presence of the new servers. Hence, it only makes sense to increase the cluster size on a version switch.

What HippoDB does *not* support
------------------------------

Anything else :-)

Structure of HippoDB
---------------------

HippoDB consists of a few separate subprojects. This does not mean it is necessarily complicated, as some of them only amount to a few lines.

* **hippodb-server** is the core of the project. It takes care of actually serving data and mantaining contacts with the cluster.
* **hippodb-client** can be used by JVM applications. Akka applications can just use a dedicated actor (HippoDB is based on Akka), while other applications will have a separate client.
* **hippodb-http** allows to serve data which is present on HippoDB through an HTTP interface. It is useful in case some applications that are not on the JVM need a way to talk to HippoDB.
* **hippodb-hbase-sync** is a map reduce job that can be used to keep in sync HBase tables with HippoDB. It produces a collection of SequenceFile on HDFS
* **hippodb-retriever** is a command line tool that retrieves the output from hbase-sync job on the local filesystem and creates the indices that are necessary for HippoDB, so that the output is ready to be served

Hence a typical deployment would have hippodb-server on each serving machine and hippodb-client used as a library from client applications. Peridocally, hippodb-hbase-sync can be run to read data from HBase and this data can be made ready on the servers using hippodb-retriever. Finally, one or more instances of hippodb-http can be deployed at will.

Since hippodb-http is based on hippodb-client, and the latter already takes care of load balancing, there should be no need to deploy hippodb-http to more that one machine and load balance between them. The workload done by hippodb-http is minimal and there will probably be no gain in having another server in front of it.

How to run
----------

There is no official deployment yet, so everything has to be run from `sbt`. The whole process will be streamlined in the future.

Servers are assumed to be identified by id. Any string will do, as long as it is used consistently.

All commands are assumed to be run from the directory containing this README. On each involved machine, one has to clone the HippoDB project.

### Step 1

Compile the MapReduce job with the command

    sbt hippodb-hbase-sync/assembly

This result in a jar file for the job under `hbase-sync/target/scala-2.10`

### Step 2

For every HBase table that you want to dump, launch the job with a command similar to

    hadoop jar hbase-sync/target/scala-2.10/hippodb-hbase-sync-assembly-1.0.jar \
      --table <table> \
      --cf <column-family> \
      --columns <col1,col2,col3> \
      --replicas <number-of-replicas> \
      --partitions <number-of-partitions> \
      --servers <server1,server2,server3> \
      --version <version-name> \
      --quorum <zookeeper-quorum> \
      --output <output-path>

The meaning of the parameters is as follows:

* `table` and `cf` are the same as in HBase; for now HippoDB only supports a single column family
* `columns`: a comma-separated list of columns to be dumped. This will become optional: if the list is missing, the whole column family will be dumped
* `replicas`: the number of times data should be replicated
* `partitions`: the number of shards to use locally on each server. `12` is a good starting point - increase it for big tables. This should be computed by HippoDB based on the amount of data automatically.
* `servers`: a comma-separated list of identifiers for the servers that will serve the data
* `version`: any string will do; used to tell apart different versions of the same table
* `quorum`: the quorum for ZooKeeper. This should also become unnecessary.
* `output`: the path on HDFS where to store the output data.

If the job completes succesfully, data will be found under `<output>/<version>/<table>`, already split between servers.

### Step 3

Servers have to retrieve data locally and index it. To do so, launch a command like

    sbt hippodb-retriever/run --source <source> --table <table>
      --version <version> --id <id> --target <target>

The meaning of the parameters is as follows:

* `table` and `version` are as in Step 2
* `id` is the local id of the server on which the command is run
* `source` is the directory on HDFS where data is to be found. It corresponds to `output` from Step 2, but should be in the form `hdfs://<nameserver>:<port>/<path>`. The port for the nameserver is usually `8020`.
* `target` is the local directory where data should be stored.

### Step 4

Configure and start the servers locally. Configuration is done by

    cp server/src/main/resources/application-local-example.conf \
      server/src/main/resources/application-local.conf

and customizing the values in that file. In particular, set the key `akka.cluster.seed-nodes`. Servers will start and try to join the cluster having at least one of this nodes. You can put a single server address there, or even the list of addresses of the whole cluster. Just make sure that the seed nodes are started first. Then, the server is started with

    sbt hippodb-server/run

Servers should recognize each other and join the cluster.

### Step 5

Now the data is available. The easiest thing to check that everything is ok is to start the HTTP server. This can be started on any machine - whether it is part of the cluster or not.

To configure the server, do

    cp http/src/main/resources/application-local-example.conf \
      http/src/main/resources/application-local.conf

and customize the values in that file. In particular, configure the key `hippo.servers` to point at one or more servers in the cluster, as in Step 4. Then the HTTP server is started with

    sbt hippodb-http/run

To check that everything is ok, try to navigate to `http://<machine>:<port>/siblings` to see the list of servers in the cluster. Here `machine` and `port` are as configured in the local configuration file.

Queries are always multiget queries, for a list of keys and columns, and can be issued to `http://<machine>:<port>/query?table=<table>&key=<key1>&key=<key2>&...&column=<column1>&column=<column2>...`.

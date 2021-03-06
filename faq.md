---
title: "Frequently Asked Questions (FAQ)"
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

The following questions are frequently asked with regard to the Flink project **in general**. If you have further questions, make sure to consult the [documentation]() or [ask the community]().

{% toc %}





## General

### Is Flink a Hadoop Project?

Flink is a data processing system and an **alternative to Hadoop's
MapReduce component**. It comes with its *own runtime* rather than building on top
of MapReduce. As such, it can work completely independently of the Hadoop
ecosystem. However, Flink can also access Hadoop's distributed file
system (HDFS) to read and write data, and Hadoop's next-generation resource
manager (YARN) to provision cluster resources. Since most Flink users are
using Hadoop HDFS to store their data, Flink already ships the required libraries to
access HDFS.

### Do I have to install Apache Hadoop to use Flink?

**No**. Flink can run **without** a Hadoop installation. However, a *very common*
setup is to use Flink to analyze data stored in the Hadoop Distributed
File System (HDFS). To make these setups work out of the box, Flink bundles the
Hadoop client libraries by default.

Additionally, we provide a special YARN Enabled download of Flink for
users with an existing Hadoop YARN cluster. [Apache Hadoop
YARN](http://hadoop.apache.org/docs/r2.2.0/hadoop-yarn/hadoop-yarn-site/YARN.html)
is Hadoop's cluster resource manager that allows use of
different execution engines next to each other on a cluster.

## Usage

### How do I assess the progress of a Flink program?

There are a multiple of ways to track the progress of a Flink program:

- The JobManager (the master of the distributed system) starts a web interface
to observe program execution. In runs on port 8081 by default (configured in
`conf/flink-config.yml`).
- When you start a program from the command line, it will print the status
changes of all operators as the program progresses through the operations.
- All status changes are also logged to the JobManager's log file.

### How can I figure out why a program failed?

- The JobManager web frontend (by default on port 8081) displays the exceptions
of failed tasks.
- If you run the program from the command-line, task exceptions are printed to
the standard error stream and shown on the console.
- Both the command line and the web interface allow you to figure out which
parallel task first failed and caused the other tasks to cancel the execution.
- Failing tasks and the corresponding exceptions are reported in the log files
of the master and the worker where the exception occurred
(`log/flink-<user>-jobmanager-<host>.log` and
`log/flink-<user>-taskmanager-<host>.log`).

### How do I debug Flink programs?

- When you start a program locally with the [LocalExecutor]({{site.docs-snapshot}}/apis/local_execution.html),
you can place breakpoints in your functions and debug them like normal
Java/Scala programs.
- The [Accumulators]({{ site.docs-snapshot }}/apis/programming_guide.html#accumulators--counters) are very helpful in
tracking the behavior of the parallel execution. They allow you to gather
information inside the program's operations and show them after the program
execution.

### What is the parallelism? How do I set it?

In Flink programs, the parallelism determines how operations are split into
individual tasks which are assigned to task slots. Each node in a cluster has at
least one task slot. The total number of task slots is the number of all task slots
on all machines. If the parallelism is set to `N`, Flink tries to divide an
operation into `N` parallel tasks which can be computed concurrently using the
available task slots. The number of task slots should be equal to the
parallelism to ensure that all tasks can be computed in a task slot concurrently.

**Note**: Not all operations can be divided into multiple tasks. For example, a
`GroupReduce` operation without a grouping has to be performed with a
parallelism of 1 because the entire group needs to be present at exactly one
node to perform the reduce operation. Flink will determine whether the
parallelism has to be 1 and set it accordingly.

The parallelism can be set in numerous ways to ensure a fine-grained control
over the execution of a Flink program. See
the [Configuration guide]({{ site.docs-snapshot }}/setup/config.html#common-options) for detailed instructions on how to
set the parallelism. Also check out [this figure]({{ site.docs-snapshot }}/setup/config.html#configuring-taskmanager-processing-slots) detailing
how the processing slots and parallelism are related to each other.

## Errors

### Why am I getting a "NonSerializableException" ?

All functions in Flink must be serializable, as defined by [java.io.Serializable](http://docs.oracle.com/javase/7/docs/api/java/io/Serializable.html).
Since all function interfaces are serializable, the exception means that one
of the fields used in your function is not serializable.

In particular, if your function is an inner class, or anonymous inner class,
it contains a hidden reference to the enclosing class (usually called `this$0`, if you look
at the function in the debugger). If the enclosing class is not serializable, this is probably
the source of the error. Solutions are to

- make the function a standalone class, or a static inner class (no more reference to the enclosing class)
- make the enclosing class serializable
- use a Java 8 lambda function.

### In Scala API, I get an error about implicit values and evidence parameters

It means that the implicit value for the type information could not be provided.
Make sure that you have an `import org.apache.flink.api.scala._` statement in your code.

If you are using flink operations inside functions or classes that take
generic parameters a TypeInformation must be available for that parameter.
This can be achieved by using a context bound:

~~~scala
def myFunction[T: TypeInformation](input: DataSet[T]): DataSet[Seq[T]] = {
  input.reduceGroup( i => i.toSeq )
}
~~~

See [Type Extraction and Serialization]({{ site.docs-snapshot }}/internals/types_serialization.html) for
an in-depth discussion of how Flink handles types.

### I get an error message saying that not enough buffers are available. How do I fix this?

If you run Flink in a massively parallel setting (100+ parallel threads),
you need to adapt the number of network buffers via the config parameter
`taskmanager.network.numberOfBuffers`.
As a rule-of-thumb, the number of buffers should be at least
`4 * numberOfTaskManagers * numberOfSlotsPerTaskManager^2`. See
[Configuration Reference]({{ site.docs-snapshot }}/setup/config.html#configuring-the-network-buffers) for details.

### My job fails early with a java.io.EOFException. What could be the cause?

The most common case for these exception is when Flink is set up with the
wrong HDFS version. Because different HDFS versions are often not compatible
with each other, the connection between the filesystem master and the client
breaks.

~~~bash
Call to <host:port> failed on local exception: java.io.EOFException
    at org.apache.hadoop.ipc.Client.wrapException(Client.java:775)
    at org.apache.hadoop.ipc.Client.call(Client.java:743)
    at org.apache.hadoop.ipc.RPC$Invoker.invoke(RPC.java:220)
    at $Proxy0.getProtocolVersion(Unknown Source)
    at org.apache.hadoop.ipc.RPC.getProxy(RPC.java:359)
    at org.apache.hadoop.hdfs.DFSClient.createRPCNamenode(DFSClient.java:106)
    at org.apache.hadoop.hdfs.DFSClient.<init>(DFSClient.java:207)
    at org.apache.hadoop.hdfs.DFSClient.<init>(DFSClient.java:170)
    at org.apache.hadoop.hdfs.DistributedFileSystem.initialize(DistributedFileSystem.java:82)
    at org.apache.flinkruntime.fs.hdfs.DistributedFileSystem.initialize(DistributedFileSystem.java:276
~~~

Please refer to the [download page]({{ site.baseurl }}/downloads.html#maven) and
the {% github README.md master "build instructions" %}
for details on how to set up Flink for different Hadoop and HDFS versions.


### My job fails with various exceptions from the HDFS/Hadoop code. What can I do?

Flink is shipping with the Hadoop 2.2 binaries by default. These binaries are used
to connect to HDFS or YARN.
It seems that there are some bugs in the HDFS client which cause exceptions while writing to HDFS
(in particular under high load).
Among the exceptions are the following:

- `HDFS client trying to connect to the standby Namenode "org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.ipc.StandbyException): Operation category READ is not supported in state standby"`
- `java.io.IOException: Bad response ERROR for block BP-1335380477-172.22.5.37-1424696786673:blk_1107843111_34301064 from datanode 172.22.5.81:50010
  at org.apache.hadoop.hdfs.DFSOutputStream$DataStreamer$ResponseProcessor.run(DFSOutputStream.java:732)`

- `Caused by: org.apache.hadoop.ipc.RemoteException(java.lang.ArrayIndexOutOfBoundsException): 0
        at org.apache.hadoop.hdfs.server.blockmanagement.DatanodeManager.getDatanodeStorageInfos(DatanodeManager.java:478)
        at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.updatePipelineInternal(FSNamesystem.java:6039)
        at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.updatePipeline(FSNamesystem.java:6002)`

If you are experiencing any of these, we recommend using a Flink build with a Hadoop version matching
your local HDFS version.
You can also manually build Flink against the exact Hadoop version (for example
when using a Hadoop distribution with a custom patch level)

### In Eclipse, I get compilation errors in the Scala projects

Flink uses a new feature of the Scala compiler (called "quasiquotes") that have not yet been properly
integrated with the Eclipse Scala plugin. In order to make this feature available in Eclipse, you
need to manually configure the *flink-scala* project to use a *compiler plugin*:

- Right click on *flink-scala* and choose "Properties"
- Select "Scala Compiler" and click on the "Advanced" tab. (If you do not have that, you probably have not set up Eclipse for Scala properly.)
- Check the box "Use Project Settings"
- In the field "Xplugin", put the path "/home/<user-name>/.m2/repository/org/scalamacros/paradise_2.10.4/2.0.1/paradise_2.10.4-2.0.1.jar"
- NOTE: You have to build Flink with Maven on the command line first, to make sure the plugin is downloaded.

### My program does not compute the correct result. Why are my custom key types
are not grouped/joined correctly?

Keys must correctly implement the methods `java.lang.Object#hashCode()`,
`java.lang.Object#equals(Object o)`, and `java.util.Comparable#compareTo(...)`.
These methods are always backed with default implementations which are usually
inadequate. Therefore, all keys must override `hashCode()` and `equals(Object o)`.

### I get a java.lang.InstantiationException for my data type, what is wrong?

All data type classes must be public and have a public nullary constructor
(constructor with no arguments). Further more, the classes must not be abstract
or interfaces. If the classes are internal classes, they must be public and
static.

### I can't stop Flink with the provided stop-scripts. What can I do?

Stopping the processes sometimes takes a few seconds, because the shutdown may
do some cleanup work.

In some error cases it happens that the JobManager or TaskManager cannot be
stopped with the provided stop-scripts (`bin/stop-local.sh` or `bin/stop-
cluster.sh`). You can kill their processes on Linux/Mac as follows:

- Determine the process id (pid) of the JobManager / TaskManager process. You
can use the `jps` command on Linux(if you have OpenJDK installed) or command
`ps -ef | grep java` to find all Java processes.
- Kill the process with `kill -9 <pid>`, where `pid` is the process id of the
affected JobManager or TaskManager process.

On Windows, the TaskManager shows a table of all processes and allows you to
destroy a process by right its entry.

Both the JobManager and TaskManager services will write signals (like SIGKILL
and SIGTERM) into their respective log files. This can be helpful for 
debugging issues with the stopping behavior.

### I got an OutOfMemoryException. What can I do?

These exceptions occur usually when the functions in the program consume a lot
of memory by collection large numbers of objects, for example in lists or maps.
The OutOfMemoryExceptions in Java are kind of tricky. The exception is not
necessarily thrown by the component that allocated most of the memory but by the
component that tried to requested the latest bit of memory that could not be
provided.

There are two ways to go about this:

1. See whether you can use less memory inside the functions. For example, use
arrays of primitive types instead of object types.

2. Reduce the memory that Flink reserves for its own processing. The
TaskManager reserves a certain portion of the available memory for sorting,
hashing, caching, network buffering, etc. That part of the memory is unavailable
to the user-defined functions. By reserving it, the system can guarantee to not
run out of memory on large inputs, but to plan with the available memory and
destage operations to disk, if necessary. By default, the system reserves around
70% of the memory. If you frequently run applications that need more memory in
the user-defined functions, you can reduce that value using the configuration
entries `taskmanager.memory.fraction` or `taskmanager.memory.size`. See the
[Configuration Reference]({{ site.docs-snapshot }}/setup/config.html) for details. This will leave more memory to JVM heap,
but may cause data processing tasks to go to disk more often.

Another reason for OutOfMemoryExceptions is the use of the wrong state backend.
By default, Flink is using a heap-based state backend for operator state in
streaming jobs. The `RocksDBStateBackend` allows state sizes larger than the
available heap space.

### Why do the TaskManager log files become so huge?

Check the logging behavior of your jobs. Emitting logging per object or tuple may be
helpful to debug jobs in small setups with tiny data sets but can limit performance
and consume substantial disk space if used for large input data.

### The slot allocated for my task manager has been released. What should I do?

If you see a `java.lang.Exception: The slot in which the task was executed has been released. Probably loss of TaskManager` even though the TaskManager did actually not crash, it 
means that the TaskManager was unresponsive for a time. That can be due to network issues, but is frequently due to long garbage collection stalls.
In this case, a quick fix would be to use an incremental Garbage Collector, like the G1 garbage collector. It usually leads to shorter pauses. Furthermore, you can dedicate more memory to
the user code by reducing the amount of memory Flink grabs for its internal operations (see configuration of TaskManager managed memory).

If both of these approaches fail and the error persists, simply increase the TaskManager's heartbeat pause by setting AKKA_WATCH_HEARTBEAT_PAUSE (akka.watch.heartbeat.pause) to a greater value (e.g. 600s).
This will cause the JobManager to wait for a heartbeat for a longer time interval before considering the TaskManager lost.

## YARN Deployment

### The YARN session runs only for a few seconds

The `./bin/yarn-session.sh` script is intended to run while the YARN-session is
open. In some error cases however, the script immediately stops running. The
output looks like this:

~~~
07:34:27,004 INFO  org.apache.hadoop.yarn.client.api.impl.YarnClientImpl         - Submitted application application_1395604279745_273123 to ResourceManager at jobtracker-host
Flink JobManager is now running on worker1:6123
JobManager Web Interface: http://jobtracker-host:54311/proxy/application_1295604279745_273123/
07:34:51,528 INFO  org.apache.flinkyarn.Client                                   - Application application_1295604279745_273123 finished with state FINISHED at 1398152089553
07:34:51,529 INFO  org.apache.flinkyarn.Client                                   - Killing the Flink-YARN application.
07:34:51,529 INFO  org.apache.hadoop.yarn.client.api.impl.YarnClientImpl         - Killing application application_1295604279745_273123
07:34:51,534 INFO  org.apache.flinkyarn.Client                                   - Deleting files in hdfs://user/marcus/.flink/application_1295604279745_273123
07:34:51,559 INFO  org.apache.flinkyarn.Client                                   - YARN Client is shutting down
~~~

The problem here is that the Application Master (AM) is stopping and the YARN client assumes that the application has finished.

There are three possible reasons for that behavior:

- The ApplicationMaster exited with an exception. To debug that error, have a
look in the logfiles of the container. The `yarn-site.xml` file contains the
configured path. The key for the path is `yarn.nodemanager.log-dirs`, the
default value is `${yarn.log.dir}/userlogs`.

- YARN has killed the container that runs the ApplicationMaster. This case
happens when the AM used too much memory or other resources beyond YARN's
limits. In this case, you'll find error messages in the nodemanager logs on
the host.

- The operating system has shut down the JVM of the AM. This can happen if the
YARN configuration is wrong and more memory than physically available is
configured. Execute `dmesg` on the machine where the AM was running to see if
this happened. You see messages from Linux' [OOM killer](http://linux-mm.org/OOM_Killer).

### My YARN containers are killed because they use too much memory

This is usually indicated my a log message like the following one:

~~~
Container container_e05_1467433388200_0136_01_000002 is completed with diagnostics: Container [pid=5832,containerID=container_e05_1467433388200_0136_01_000002] is running beyond physical memory limits. Current usage: 2.3 GB of 2 GB physical memory used; 6.1 GB of 4.2 GB virtual memory used. Killing container.
~~~

In that case, the JVM process grew too large. Because the Java heap size is always limited, the extra memory typically comes from non-heap sources:

  - Libraries that use off-heap memory. (Flink's own off-heap memory is limited and taken into account when calculating the allowed heap size.)
  - PermGen space (strings and classes), code caches, memory mapped jar files
  - Native libraries (RocksDB)

You can activate the [memory debug logger](https://ci.apache.org/projects/flink/flink-docs-release-1.0/setup/config.html#memory-and-performance-debugging) to get more insight into what memory pool is actually using up too much memory.


### The YARN session crashes with a HDFS permission exception during startup

While starting the YARN session, you are receiving an exception like this:

~~~
Exception in thread "main" org.apache.hadoop.security.AccessControlException: Permission denied: user=robert, access=WRITE, inode="/user/robert":hdfs:supergroup:drwxr-xr-x
  at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.check(FSPermissionChecker.java:234)
  at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.check(FSPermissionChecker.java:214)
  at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkPermission(FSPermissionChecker.java:158)
  at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.checkPermission(FSNamesystem.java:5193)
  at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.checkPermission(FSNamesystem.java:5175)
  at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.checkAncestorAccess(FSNamesystem.java:5149)
  at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.startFileInternal(FSNamesystem.java:2090)
  at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.startFileInt(FSNamesystem.java:2043)
  at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.startFile(FSNamesystem.java:1996)
  at org.apache.hadoop.hdfs.server.namenode.NameNodeRpcServer.create(NameNodeRpcServer.java:491)
  at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolServerSideTranslatorPB.create(ClientNamenodeProtocolServerSideTranslatorPB.java:301)
  at org.apache.hadoop.hdfs.protocol.proto.ClientNamenodeProtocolProtos$ClientNamenodeProtocol$2.callBlockingMethod(ClientNamenodeProtocolProtos.java:59570)
  at org.apache.hadoop.ipc.ProtobufRpcEngine$Server$ProtoBufRpcInvoker.call(ProtobufRpcEngine.java:585)
  at org.apache.hadoop.ipc.RPC$Server.call(RPC.java:928)
  at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2053)
  at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2049)
  at java.security.AccessController.doPrivileged(Native Method)
  at javax.security.auth.Subject.doAs(Subject.java:396)
  at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1491)
  at org.apache.hadoop.ipc.Server$Handler.run(Server.java:2047)

  at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
  at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:39)
  at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:27)
  at java.lang.reflect.Constructor.newInstance(Constructor.java:513)
  at org.apache.hadoop.ipc.RemoteException.instantiateException(RemoteException.java:106)
  at org.apache.hadoop.ipc.RemoteException.unwrapRemoteException(RemoteException.java:73)
  at org.apache.hadoop.hdfs.DFSOutputStream.newStreamForCreate(DFSOutputStream.java:1393)
  at org.apache.hadoop.hdfs.DFSClient.create(DFSClient.java:1382)
  at org.apache.hadoop.hdfs.DFSClient.create(DFSClient.java:1307)
  at org.apache.hadoop.hdfs.DistributedFileSystem$6.doCall(DistributedFileSystem.java:384)
  at org.apache.hadoop.hdfs.DistributedFileSystem$6.doCall(DistributedFileSystem.java:380)
  at org.apache.hadoop.fs.FileSystemLinkResolver.resolve(FileSystemLinkResolver.java:81)
  at org.apache.hadoop.hdfs.DistributedFileSystem.create(DistributedFileSystem.java:380)
  at org.apache.hadoop.hdfs.DistributedFileSystem.create(DistributedFileSystem.java:324)
  at org.apache.hadoop.fs.FileSystem.create(FileSystem.java:905)
  at org.apache.hadoop.fs.FileSystem.create(FileSystem.java:886)
  at org.apache.hadoop.fs.FileSystem.create(FileSystem.java:783)
  at org.apache.hadoop.fs.FileUtil.copy(FileUtil.java:365)
  at org.apache.hadoop.fs.FileUtil.copy(FileUtil.java:338)
  at org.apache.hadoop.fs.FileSystem.copyFromLocalFile(FileSystem.java:2021)
  at org.apache.hadoop.fs.FileSystem.copyFromLocalFile(FileSystem.java:1989)
  at org.apache.hadoop.fs.FileSystem.copyFromLocalFile(FileSystem.java:1954)
  at org.apache.flinkyarn.Utils.setupLocalResource(Utils.java:176)
  at org.apache.flinkyarn.Client.run(Client.java:362)
  at org.apache.flinkyarn.Client.main(Client.java:568)
~~~

The reason for this error is, that the home directory of the user **in HDFS**
has the wrong permissions. The user (in this case `robert`) can not create
directories in his own home directory.

Flink creates a `.flink/` directory in the users home directory
where it stores the Flink jar and configuration file.


### My job is not reacting to a job cancellation?

Flink is canceling a job by calling the `cancel()` method on all user tasks. Ideally,
the tasks properly react to the call and stop what they are currently doing, so that 
all threads can shut down.

If the tasks are not reacting for a certain amount of time, Flink will start interrupting
the thread periodically.

The TaskManager logs will also contain the current stack of the method where the user 
code is blocked. 


## Features

### What kind of fault-tolerance does Flink provide?

For streaming programs Flink has a novel approach to draw periodic snapshots of the streaming dataflow state and use those for recovery.
This mechanism is both efficient and flexible. See the documentation on [streaming fault tolerance]({{ site.docs-snapshot }}/internals/stream_checkpointing.html) for details.

For batch processing programs Flink remembers the program's sequence of transformations and can restart failed jobs.


### Are Hadoop-like utilities, such as Counters and the DistributedCache supported?

[Flink's Accumulators]({{ site.docs-snapshot }}/apis/programming_guide.html#accumulators--counters) work very similar like
Hadoop's counters, but are more powerful.

Flink has a [Distributed Cache](https://github.com/apache/flink/tree/master/flink-core/src/main/java/org/apache/flink/api/common/cache/DistributedCache.java) that is deeply integrated with the APIs. Please refer to the [JavaDocs](https://github.com/apache/flink/tree/master/flink-java/src/main/java/org/apache/flink/api/java/ExecutionEnvironment.java#L831) for details on how to use it.

In order to make data sets available on all tasks, we encourage you to use [Broadcast Variables]({{ site.docs-snapshot }}/apis/programming_guide.html#broadcast-variables) instead. They are more efficient and easier to use than the distributed cache.

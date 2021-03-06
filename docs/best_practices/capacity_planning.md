## Setting Up Your Cluster

The first rule to observe when planning is to know that no one size fits all capacity planning, as different customers have different requirement and the best solution is to create a plan that is most suitable for your environment. Therefore, capacity planning has been a critical component of successful implementations.

<a id="cores"></a>
### Concurrency and Management of Cores

Executing queries or code in SnappyData results in the creation of one or more Spark jobs. Each Spark job has multiple tasks. The number of tasks is determined by the number of partitions of the underlying data.  
Concurrency in SnappyData is tightly bound with the capacity of the cluster, which means, the number of cores available in the cluster determines the number of concurrent tasks that can be run. 

The default setting is </br>
**CORES = 2 X number of cores on a machine**. 

`spark.executor.cores` is used to override a number of cores per server.

For example, for a cluster with 2 servers running on two different machines with  4 CPU cores each, a maximum number of tasks that can run concurrently in the cluster would be 16. </br> 
If a table has 17 partitions (buckets, for row or column tables), a scan query on this table would create 17 tasks. This means, 16 tasks would run concurrently and the last task would run when one of these 16 tasks have finished execution. 

SnappyData uses an optimization method which clubs multiple partitions on a single machine to form a single partition when there are fewer cores available. This reduces the overhead of scheduling partitions. 

In SnappyData, multiple queries can be executed concurrently, if they were submitted by different threads or different jobs. For concurrent queries, SnappyData uses fair scheduling to manage the available resources such that all the queries get fair resources. 
 
For example: In the image below, 6 cores are available on 3 systems, and 2 jobs have 4 tasks each. Because of fair scheduling, both jobs get 3 cores and hence three tasks are per job execute concurrently.

Pending tasks have to wait for the completion of the current tasks and are assigned to the core that is first available on completion of the current tasks.

When you add more servers to SnappyData, the processing capacity of the system increases in terms of available cores. Thus, more cores are available so more tasks can concurrently execute.

![Concurrency](/../../Images/core_concurrency.png)

We recommend using **2 X number of cores** on a machine. If more than one server is running on a machine, the cores should be divided accordingly and specified using the `spark.cores.executors` property.

<a id="heap"></a>
### Memory Management: Heap and Off-Heap 

SnappyData is a Java application and by default supports on-heap storage. At the same time, to improve the performance for large chunks of data, it also supports off-heap storage. Column tables can be stored on off-heap storage as they are large chunks of columns stored as byte arrays. 

We recommend using off-heap storage for column tables. Row tables are always stored on on-heap. The [memory-size](/../../configuring_cluster/property_description#memory-size) and [heap-size](/../../configuring_cluster/property_description#heap-size) properties control the off-heap and on-heap size behavior of the SnappyData server process. 

The memory pool (off-heap and on-heap) available in SnappyData's cluster is divided into two parts – Execution and Storage memory pool. The storage memory pool as the name indicates is for the table storage. The amount of memory that is available for storage is 50% of the total memory but it can grow to 75% if the execution memory is unused. 

This can be altered by specifying the `spark.memory.storageFraction` property. But, we do not recommend changing this setting. 

<!--Shall we talk about the reserved memory? <mark>Sumedh </mark>-->

SnappyData tables are by default configured for eviction which means, when there is memory pressure, the tables are evicted to disk. This impacts performance to some degree and hence we recommend sizing your VM before you begin. 

![On-Heap and Off-Heap](/../../Images/on-off-heap.png)

<!-- Default values for sizing the VM <mark> Sumedh</mark>-->

<a id="ha-consideration"></a>
### HA Considerations

High availability options are available for all the SnappyData components. 

**Lead HA** </br> 
SnappyData supports secondary lead nodes. If the primary lead becomes unavailable, one of  the secondary lead nodes takes over immediately. 
Setting up the secondary lead node is highly recommended because the system cannot function if the lead node is unavailable. Currently, the queries that are executing when the primary lead becomes unavailable, is not re-tried and has to be resubmitted.

**Locator **</br>  
SnappyData supports multiple locators in the cluster for high availability. 
It is recommended to set up multiple locators as if a locator becomes unavailable, the cluster continues to be available, but new members cannot join the cluster.
With multiple locators, clients notice nothing and the failover recovery is completely transparent.

**DataServer**</br> 
SnappyData supports redundant copies of data for fault tolerance. A table can be configured to store redundant copies of the data.  So, if a server is unavailable, and if there is a redundant copy available on some other server, the tasks are automatically retried on those servers. This is totally transparent to the user. 
However, the redundant copies double the memory requirements. If there are no redundant copies and a server with some data goes down, the execution of the queries fail and PartitionOfflineException is reported. The execution does not begin until that server is available again. 

###  SnappyData Settings 
<a id="buckets"></a>
#### Buckets

Bucket is the unit of partitioning for SnappyData tables. The data is distributed evenly across all the buckets. When a new server joins or an existing server leaves the cluster, the buckets are moved around for rebalancing. 

The number of buckets should be set according to the table size. By default, there are 113 buckets for a table. 
If there are more buckets in a table than required, it means there is fewer data per bucket. For column tables, this may result in reduced compression that SnappyData achieves with various encodings. 
Similarly, if there are not enough buckets in a table, not enough partitions are created while running a query and hence cluster resources are not used efficiently.
Also, if the cluster is scaled at a later point of time rebalancing may not be optimal.

For column tables, we recommend setting a number of buckets such that each bucket has at least 100-150 MB of data.  

#### member-timeout

SnappyData efficiently uses CPU for running OLAP queries. This results in 100% of the CPU being used when the cluster is running to its full capacity. In such cases, due to the amount of garbage generated and CPU being occupied, the system may experience a minor freeze. These freezes are rare and can also be minimized by the setting the off-heap property. </br>
However, for such cases, the `member-timeout` should be increased to a higher value. This would ensure that the members are not thrown out of the cluster in case of a system freeze. 

The default value of member-timeout is: 5 sec. 

``` 
DistributionConfig.DEFAULT_MEMBER_TIMEOUT=5000
```

#### spark.local.dir  

SnappyData writes table data on disk.  By default, the disk location that SnappyData uses is the directory specified using `-dir` option, while starting the member. 
SnappyData also uses temporary storage for storing intermediate data. The amount of intermediate data depends on the type of query and can be in the range of the actual data size. 
To achieve better performance, we recommend storing temporary data on a different disk than the table data. This can be done by setting the `spark.local.dir` parameter.

#### -XX:-DontCompileHugeMethods

By default, SnappyData enables Just In Time (JIT) compilation for large methods (> 8K bytecode) in the cluster mode. This is set using the `-XX:-DontCompileHugeMethods=false` option.</br> 
For generated code of query plans that refer to relatively large number of columns (> 15), this can provide a significant performance boost. </br>
This option is recommended to be explicitly added when using the product in local or Smart Connector mode as:

```bash
spark-submit --conf spark.driver.extraJavaOptions="-XX:-DontCompileHugeMethods=false" --conf spark.executor.extraJavaOptions="-XX:-DontCompileHugeMethods=false" --master ...
```

<a id="os_setting"></a>
###  Operating System Settings 

For best performance, the following operating system settings are recommended on the lead and server nodes:

**Ulimit** </br> 
The server and lead node machines need to be configured for higher limits of open files and threads/processes. The recommended settings are:

Spark and SnappyData spawn a number of threads and sockets for concurrent/parallel processing so the server and lead node machines may need to be configured for higher limits of open files and threads/processes. </br>On a Linux system this is configured in the */etc/security/**limits.conf*** file.
</br>A minimum of 8192 is recommended for open file descriptors limit and nproc limit to be greater than 128K. 
</br>To change the limits of these settings for a user, /etc/security/limits.conf file needs to be updated. A typical limits.conf used for SnappyData servers and leads looks like: 

```
ec2-user          hard    nofile      163840 
ec2-user          soft    nofile      16384
ec2-user          hard    nproc       unlimited
ec2-user          soft    nproc       524288
ec2-user          hard    sigpending  unlimited
ec2-user          soft    sigpending  524288
```
* `ec2-user` is the user running SnappyData.	


**OS Cache Size**</br> 
When there is lot of disk activity especially during table joins and during eviction, the process may experience GC pauses. To avoid such situations, we recommend reducing the OS cache size by specifying a lower dirty ratio and less expiry time of the dirty pages.</br> 
The following are the typical configuration to be done on the machines that are running SnappyData processes. 

```
sudo sysctl -w vm.dirty_background_ratio=2
sudo sysctl -w vm.dirty_ratio=4
sudo sysctl -w vm.dirty_expire_centisecs=2000
sudo sysctl -w vm.dirty_writeback_centisecs=300
```

**Swap File** </br> 
Since moderen OSes do lazy allocation, we have observed that despite setting `-Xmx` and `-Xms` settings, at runtime, the operating system may fail to allocate new pages to the JVM. This can result in process going down.</br>
We recommend setting swap space on your system using the following commands.

```
# sets a swap space of 32 GB
sudo dd if=/dev/zero of=/var/swapfile.1 bs=1M count=32768
sudo chmod 600 /var/swapfile.1
sudo mkswap /var/swapfile.1
sudo swapon /var/swapfile.1
```


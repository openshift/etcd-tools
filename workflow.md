# Generic

Latency (network, storage) should be stable whole time as excessive peaks could mean problem for ETCD (and therefor whole cluster). Even only few minutes lasting peak in latency could mean timeout on ETCD, API and oauth pods and cause inability to login to cluster.

**ETCD performance depends mainly on storage, CPU and network latency, but be aware it can be combination of all of them and focusing on single part is bad idea!**


## Master node size

Customer should understand that

- masters are most important nodes and therefor they should have best resources (CPU, storage, networking)
- masters should have dedicated resources, mixing master and worker workload can cause cluster stability issues
- with Openshift, only 3 masters are supported no matter how big cluster is (and bigger the cluster, more resources are needed on masters)
- for best performance, virtualized masters should have dedicated storage/LUN and be hosted on dedicated hypervisor (not shared with other VMs)

There is no exact formula, but masters should be sized depending on load (mainly load on API and ETCD caused by operators or pipelines).

What to do:

Examine size of cluster, apart from regular sizes *you could find also non-common setup or size which can be confusing* (small cluster with too many resources or opposite).

Example:

1. 3 worker node running heavy DB can be more overloaded than same size cluster running light nginx containers.
2. single node (SNO) cluster running on node with 64 CPUs is stronger than 3 node cluster with 8 CPU nodes.
3. cluster with 120 workers and no infra nodes

Check number of operators and pods and asses if load is not too big for size of the cluster/nodes.



## Cluster size

average sizing would be

```
small - SNO, Edge or up to 5 workers and no infra
medium - 10-20 workers and few infra
large - 20+ workers and 6+ infras
huge - 50-100+ workers and 10-20+ infras
```

**small cluster with minimum or medium resources is good only for local development or demo or SNO/Edge scenarios**

### CPU

Check the load (number of installed operators/pipelines and how heavy they are) and check if master nodes have enough CPU.

```
minimum - 4 CPU, good only for local development or demo
medium - 8 CPU, good for small to medium cluster with average load
large - 16 CPU, good for small to medium cluster with heavy load (or too many operators, pipelines, etc..) or large cluster with average load
huge - 20+ CPU, good for large cluster with heavy load
```



### Memory

RAM depends heavily on installed operators (for example logging can be quite heavy on CPU and RAM).



# CPU

## iowait 

values should be below 4.0. Values over 8.0 are alarming. Always check sizing of nodes if they have enough CPU/RAM if you see high iowait.




# Storage


## Generic storage behaviour

There's always will be pros and cons as you cannot have storage that would have best concurrent IOPS but also sequential, or super high IOPS and super low latency. Customer should find balance between DB/ETCD related performance and generic (worker load) performance.


## Metrics


## IOPS

required fsync sequential IOPS:

* 50 - minimum, local development
* 300 - medium cluster with average load
* 500 - medium or large cluster
* 800+ - large cluster with heavy load

[fio suite]

Importance of data is in following order. Make sure you get data with highest priority first and examine them first. 

1. Fsync latency and fsync sequential IOPS (is storage tweaked for ETCD?) [How to graph etcd metrics using Prometheus to gauge Etcd performance in OpenShift](https://access.redhat.com/solutions/5489721)
2. LibIAO sequential IOPS  (is storage tweaked for sequential IO in general?)
3. Random, concurrent IOPS  (while being tweaked for sequential IOPS, how storage can handle concurrent?)

Never make conclusion only from one metric alone, but rather look at combination of data and importance/priority of the data.
**ETCD metrics are most important to see how data change over time. Never make conclusion from data that describe only one specific moment!**

Example of small/medium cluster:

| fsync seq.     | libiao seq.     | random/concurrent    | outcome | solution                                                                                                                      |
|----------------|-----------------|---------------|---------|-------------------------------------------------------------------------------------------------------------------------------|
| IOPS below 300 | IOPS below 1000 | 10k+ IOPS     | BAD     | storage is optimized for concurrent IOPS but ETCD requires sequential                                                         |
| 300-600 IOPS   | 1500-2500 IOPS  | below 10k     | GOOD    |                                                                                                                               |
| 900+ IOPS      | 4000-6000+ IOPS       | very low IOPS | BAD     | storage is optimized too much for ETCD but not for other things. High concurrent IO could degrade sequential IO a lot. |

**IMPORTANT:** all numbers in example are just rough estimates and shouldn't be taken as exact threshold. Don't focus just on numbers but also on gap between different values.


## Latency



### How to read fio/fio_suite output

```
$ oc debug node/<master_node>
  [...]
  sh-4.4# chroot /host bash
  podman run --privileged --volume /var/lib/etcd:/test quay.io/peterducai/openshift-etcd-suite:latest fio
```

<pre>
```
cleanfsynctest: (groupid=0, jobs=1): err= 0: pid=89: Tue Sep 27 16:39:22 2022
  <b>write: IOPS=230</b>, BW=517KiB/s (529kB/s)(22.0MiB/43595msec); 0 zone resets           <i><--- fsync sequential IOPS</i>
    clat (usec): min=4, max=37506, avg=63.37, stdev=393.00
     lat (usec): min=4, max=37508, avg=64.45, stdev=393.12
    clat percentiles (usec):
     |  1.00th=[    7],  5.00th=[   16], 10.00th=[   18], 20.00th=[   20],
     | 30.00th=[   25], 40.00th=[   27], 50.00th=[   31], 60.00th=[   42],
     | 70.00th=[   63], 80.00th=[   88], 90.00th=[  122], 95.00th=[  143],
     | 99.00th=[  334], 99.50th=[  717], 99.90th=[ 1369], 99.95th=[ 1516],
     | 99.99th=[ 6652]
   bw (  KiB/s): min=   49, max= 1105, per=99.86%, avg=516.54, stdev=283.00, samples=87
   iops        : min=   22, max=  492, avg=230.16, stdev=125.97, samples=87
  lat (usec)   : 10=2.22%, 20=19.09%, 50=43.13%, 100=20.00%, 250=14.21%
  lat (usec)   : 500=0.59%, 750=0.28%, 1000=0.20%
  lat (msec)   : 2=0.24%, 10=0.02%, 50=0.01%
  fsync/fdatasync/sync_file_range:
    sync (usec): min=1245, max=293908, avg=4270.40, stdev=6256.20
    sync percentiles (usec):
     |  1.00th=[ 1532],  5.00th=[ 1811], 10.00th=[ 1926], 20.00th=[ 2180],
     | 30.00th=[ 2704], 40.00th=[ 3130], 50.00th=[ 3294], 60.00th=[ 3490],
     | 70.00th=[ 3785], 80.00th=[ 4359], 90.00th=[ 5538], 95.00th=[ 6456],
     | <b>99.00th=[38011]</b>, 99.50th=[43254], <b>99.90th=[62653]</b>, 99.95th=[65799],     <i><--- 99.0th and 99.9th percentile that should be below 10k</i>
     | 99.99th=[73925]
```
</pre>

how to read IOPS on other tests

<pre>
```

LIBAIO is generic sequential test and <b>doesn't exactly emulate how ETCD works</b>, rather emulates generic sequential IO.

[ SEQUENTIAL IOPS TEST ] - [ libaio engine SINGLE JOB, 70% read, 30% write]

--------------------------
1GB file transfer:
  read: IOPS=10.3k, BW=40.3MiB/s (42.2MB/s)(471MiB/11683msec)
  write: IOPS=4444, BW=17.4MiB/s (18.2MB/s)(203MiB/11683msec); 0 zone resets
SEQUENTIAL WRITE IOPS: 4444                              <i><--- <b>4.4k is more than 30% of read IOPS so it's OK</b></i>
SEQUENTIAL READ IOPS: 10000                              <i><--- 10k is pretty high number</i>
--------------------------

<i>EXAMPLE2:</i>

  read: IOPS=10.3k, BW=40.3MiB/s (42.2MB/s)(471MiB/11683msec)
  write: IOPS=4444, BW=17.4MiB/s (18.2MB/s)(203MiB/11683msec); 0 zone resets
SEQUENTIAL WRITE IOPS: 200                              <i><--- <b>200 is less than 30% of 2500 read IOPS so it's bad..</b> also 200 is pretty low number</i>
SEQUENTIAL READ IOPS: 2500                              <i><--- 2.5k is fine, but obviously <b>storage cannot keep up with writing, while reading</b></i>
--------------------------

<i>EXAMPLE3:</i>

  read: IOPS=10.3k, BW=40.3MiB/s (42.2MB/s)(471MiB/11683msec)
  write: IOPS=4444, BW=17.4MiB/s (18.2MB/s)(203MiB/11683msec); 0 zone resets
SEQUENTIAL WRITE IOPS: 250                              <i><--- 250 is close to 30% of 800 read IOPS so it's OK.. but 250 is pretty low number for medium or large cluster</i>
SEQUENTIAL READ IOPS: 800                               <i><--- 800 is <b>not enough for serious workload</b></i>
--------------------------

...
--------------------------
200MB file transfer:
  read: IOPS=13.8k, BW=53.7MiB/s (56.3MB/s)(140MiB/2608msec)
  write: IOPS=5881, BW=23.0MiB/s (24.1MB/s)(59.9MiB/2608msec); 0 zone resets
SEQUENTIAL WRITE IOPS: 5881
SEQUENTIAL READ IOPS: 13000
--------------------------

-- [ libaio engine SINGLE JOB, 30% read, 70% write] --

--------------------------
200MB file transfer:
  read: IOPS=6517, BW=25.5MiB/s (26.7MB/s)(60.2MiB/2366msec)
  write: IOPS=15.1k, BW=59.1MiB/s (61.9MB/s)(140MiB/2366msec); 0 zone resets
SEQUENTIAL WRITE IOPS: 15000
SEQUENTIAL READ IOPS: 6517
--------------------------

--------------------------
1GB file transfer:
  read: IOPS=5893, BW=23.0MiB/s (24.1MB/s)(68.7MiB/2986msec)
  write: IOPS=13.7k, BW=53.7MiB/s (56.3MB/s)(160MiB/2986msec); 0 zone resets
SEQUENTIAL WRITE IOPS: 13000
SEQUENTIAL READ IOPS: 5893
```
</pre>

<pre>
```
[ RANDOM IOPS TEST ] - REQUEST OVERHEAD AND SEEK TIMES] ---
This job is a latency-sensitive workload that stresses per-request overhead and seek times. Random reads.


1GB file transfer:
  read: IOPS=55.1k, BW=215MiB/s (226MB/s)(1024MiB/4757msec)
--------------------------
RANDOM IOPS: 55000
--------------------------

200MB file transfer:
  read: IOPS=55.1k, BW=215MiB/s (226MB/s)(200MiB/929msec)
--------------------------
RANDOM IOPS: 55000
--------------------------
```
</pre>


OUTCOME:

- fsync 230 IOPS are OK for small cluster but not for medium
- fsync latency of 62ms is too high (should be below 10k)
- libiao IOPS values are super good
- concurrent/random IOPS are huge .. 55k, while we don't need such high numbers at all.


FYI this example is from Thinkapd with NVMe and running CRC (without CRC numbers are even better).

**IMPORTANT: Each storage behaves differently under different conditions and you should figure out which part of storage should be tweak (by customer/vendor)**



## etcd_disk_wal_fsync_duration 99th and 99.9th

should be lower than 10ms

```
>2ms = superb, probably NVMe on baremetal or AWS with io1 disk and 2000 IOPS set.
>5ms = great, usually well performing virtualized platform
5-7ms = OK
8-10ms = close to threshold, NOT GOOD if any peaks occur
```

> some versions may suggest that threshold is 20ms. Still, check docs and evaluate how many percents close to threshold value is and asses performance risk. Values above 15ms are also not good as they are close to threshold of 20ms.

> Usually when 99th is close to threshold, we will see 99.9th going above threshold, which means storage can barely provide required performance (for ETCD) and it's really better when 99.0th is below 10ms.


## etcd_disk_backend_commit_duration 99th

should be lower than 25ms

# Network

Big network latency and packet drops can also bring an unreliable etcd cluster state, so network health values (RTT and packet drops) should be monitored. 

## RX/TX errors and dropped packets

<pre>
```
ip -s link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    RX:  bytes packets errors dropped  missed   mcast           
          8296      94      0       0       0       0 
    TX:  bytes packets errors dropped carrier collsns           
          8296      94      0       0       0       0 
2: enp0s31f6: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state UP mode DORMANT group default qlen 1000
    link/ether 20:1e:88:99:df:2c brd ff:ff:ff:ff:ff:ff
    RX:  bytes packets errors <b>dropped</b>  missed   mcast     <i><--- check RX/TX errors and dropped packets     </i> 
    1993837469 1821202      0      <b>22</b>       0       0 
    TX:  bytes packets errors dropped carrier collsns           
     171129355  506637      0       0       0       0 
```
</pre>


## etcd_network_peer_round_trip_time 99th 

should be lower than 50ms. Values of 40+ mean network latency is close to threshold and any peak could degrade ETCD performance.






# ETCD size

## Object count

* small cluster - even 1-2k of objects could cause issues. Cluster should be extended to medium one for such workload.
* medium cluster - ~8k of objects could cause issues, having huge secrets/keys could mean problems even with lower number (~6k)
* large cluster
* huge cluster - with too heavy load and object count 10k+ it could mean that in future load will reach limits and have to be split onto several smaller clusters

You can get object count with

```
$ oc project openshift-etcd
oc get pods
oc rsh <etcd pod>
> etcdctl get / --prefix --keys-only | sed '/^$/d' | cut -d/ -f3 | sort | uniq -c | sort -rn
```

or with [etcd_analyzer.sh](https://github.com/peterducai/etcd-tools/blob/main/etcd-analyzer.sh)

With [cleanup_analyzer.sh](https://github.com/peterducai/etcd-tools/blob/main/cleanup-analyzer.sh) you can find out excessive number of inactive objects (images, deployments, etc..)


## Object size

If secret holds huge token, certifikate or SSH key, it might get performance problem even with less secrets than 8k (on small to medium cluster).
Check also namespaces for big amount of objects (mainly secrets). User namespace with excessive number (30+) of secrets should be ideally cleaned up.

```
oc get secrets -A --no-headers | awk '{ns[$1]++}END{for (i in ns) print i,ns[i]}'
```


<!-- ## FAQ

> Our own etcd is a little different due to rhel and some sidecar binary:

https://github.com/openshift/etcd/blob/openshift-4.14/Dockerfile.rhel

https://github.com/openshift/etcd/blob/openshift-4.14/Dockerfile.art

> When running “etcdctl endpoint health”, what does the “successfully commited proposal: took = xxxx ms” metric represents?

health is entirely implemented on the [client](https://github.com/etcd-io/etcd/blob/release-3.5/etcdctl/ctlv3/command/ep_command.go#L126-L131)
It's a simple linearized read request that goes through raft over the network. As it's a get, the apply doesn't go through the disk.

> There are no supported/documented/tested methods for moving OCP VMs on vsphere between datastores. 

The only method we officially recommend or support would be a reinstallation. With this being considered a production cluster I understand that likely is not an option. It was mentioned that previously you have migrated the backing storage for datastores at the Netapp side in a method that is transparent to both OCP and Vsphere. That likely will be your best bet. Just make sure to test thoroughly as this is not something we test ourselves.  -->
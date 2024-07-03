# We compared ScyllaDB and Memcached and… We lost?

_ScyllaDB would like to publicly acknowledge and thank_ [_dormando_](https://www.dormando.me/) _(memcached maintainer) and_ [_Danny Kopping_](https://www.linkedin.com/in/dannykopping/) _for their contributions, support and patience (!!!) on which this work got inspired._

## About this project

Engineers behind ScyllaDB – the database for predictable performance at scale – joined forces with Memcached maintainer dormando to compare both technologies head-to-head, in a collaborative vendor-neutral way.

The results reveal that both Memcached and ScyllaDB maximized disks and network bandwidth while being stressed under similar conditions, sustaining similar performance numbers overall.

While ScyllaDB required data modeling changes to fully saturate the network throughput, Memcached required additional IO threads to saturate disk I/O. Although ScyllaDB showed better latencies when compared to Memcached pipelined requests to disk, Memcached latencies were better for individual requests. Following up on those results, we analyzed the architectural differences behind those solutions, as well as discussed the tradeoffs involved in each.

If you already read the blog, skip ahead to the additional details added here:

* [Setup/configuration details](./#setup)
* [Extended discussion of tests and results](./#tests-and-results)
* Q and A

## Why have we done this?

First and foremost, ScyllaDB invested lots of time and engineering resources optimizing our database to deliver predictable low latencies for real-time data-intensive applications. ScyllaDB’s [shard-per-core](https://www.scylladb.com/product/technology/shard-per-core-architecture/), shared-nothing architecture, [userspace I/O scheduler](https://www.scylladb.com/2021/09/15/what-weve-learned-after-6-years-of-io-scheduling/) and [internal cache](https://www.scylladb.com/2024/01/08/inside-scylladbs-internal-cache/) implementation (fully bypassing the Linux page cache) are some notable examples of such optimizations.

Second: performance converges over time. In-memory caches have been (for a long time) regarded as one of the fastest infrastructure components around. Yet, it's been a few years now since caching solutions started to look into the realm of flash disks. These initiatives obviously pose an interesting question: **If an in-memory cache can rely on flash storage, then why can't a persistent database also work as a cache?**

Third: We previously discussed [7 Reasons Not to Put a Cache in Front of Your Database](https://www.scylladb.com/2017/07/31/database-caches-not-good/), as well as exemplified it with organizations which have successfully replaced their caches with ScyllaDB at [Replacing Your Cache with ScyllaDB](https://lp.scylladb.com/wbn-replacing-your-cache-registration).

Fourth: Danny Kooping gave us an enlightening talk at P99 CONF titled [Cache Me If You Can](https://www.p99conf.io/session/cache-me-if-you-can-how-grafana-labs-scaled-up-their-memcached-42x-cut-costs-too/), where he explained how Memcached Extstore helped Grafana Labs scale their cache footprint 42x while driving costs down.

And finally, despite the (valid) criticism that performance benchmarks receive, they still play an important role for driving innovation. It is a useful resource for Engineers seeking after in-house optimization opportunities .

Now onto the comparison.&#x20;

## Setup

### Instances

Tests were carried out using the following instance types in AWS:

* **Loader**: c7i.16xlarge (64 vCPUs, 128GB RAM)
* **Memcached**: i4i.4xlarge (16 vCPUs, 128GB RAM, 3.75TB NVMe)
* **ScyllaDB**: i4i.4xlarge (16 vCPUs, 128GB RAM, 3.75TB NVMe)

[All instances can deliver _up to_ 25Gbps of network bandwidth.](#user-content-fn-1)[^1]

### Optimizations and Settings

To overcome potential bottlenecks, the following optimizations & settings were applied:

* **AWS side**: All instances were used a Cluster placement strategy, following [AWS Docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html):

> "This strategy enables workloads to achieve the low-latency network performance necessary for tightly-coupled node-to-node communication that is typical of high-performance computing (HPC) applications."

* **Memcached**: Version 1.6.25, compiled with Extstore enabled. Except where denoted, run with 14 threads, pinned to specific CPUs. The remaining 2 vCPUs were assigned to CPU 0 (core & HT sibling) to handle Network IRQs, as specified by the sq\_split mode in [seastar perftune.py](https://github.com/scylladb/seastar/blob/master/scripts/perftune.py). CAS operations were disabled to save space on per-item overhead. The full command line arguments were:

> taskset -c 1-7,9-15 /usr/local/memcached/bin/memcached -v -A -r -m 114100 -c 4096 --lock-memory --threads 14 -u scylla -C

* **ScyllaDB**: Default settings as configured by ScyllaDB Enterprise 2024.1.2 AMI (ami-id: _ami-018335b47ba6bdf9a_) in a i4i.4xlarge. This includes the same CPU pinning settings as described earlier to memcached.

### Stressors

For memcached loaders, we used [mcshredder](https://github.com/memcached/mcshredder), part of memcached's official testing suite. Profiles used can be found in [fee-mendes/shredders](https://github.com/fee-mendes/shredders/tree/main) Github repository as applicable to our testing.

For ScyllaDB, we used [cassandra-stress](https://opensource.docs.scylladb.com/stable/operating-scylla/admin-tools/cassandra-stress.html), as shipped with ScyllaDB, and specified comparable workloads as the ones used for memcached.



## Tests and Results

**Note:** <mark style="color:purple;">For a summary of the tests and results, see the blog \<FIXME>.</mark>

ScyllaDB and Memcached tests were split across two groups: In-Memory and On-disk.&#x20;

We start by comparing both solutions' cache footprint and how they performed under load during a read-only workload.&#x20;

Past the cache-only part, we then introduce [Extstore](https://github.com/memcached/memcached/wiki/Extstore) to Memcached, a "_Feature for extending memcached's memory space onto flash (or similar) storage_".

### Cache Footprint

The more items you can fit into RAM, the better your chance of getting cache hits. More cache hits result in significantly faster access than going to disk. Ultimately, that improves latency.

RAM is also significantly more expensive than a GB of Flash Storage in Cloud environments. Danny Kopping estimated a cost of roughly USD$4.43/GB for RAM on GCP n2-standard machines during his talk, whereas the price for Flash Storage was down to USD$0.08/GB.

Memcached [design philosophy](https://github.com/memcached/memcached/wiki/Overview#design-philosophy) is simple: It is a key-value store with no concept of synchronization, broadcasting or replication. The more servers you have, the more caching space you get. Although you could technically implement replication on your own, the best approach would be to simply [failover](https://github.com/memcached/memcached/wiki/ConfiguringClient#failure-or-failover) (with some caveats) instead and tolerate a few cache misses during a failure situation.

ScyllaDB lives on the other side of the spectrum: It is a wide-column persistent storage, with synchronization and replication features built into its architecture. In that regard, more servers doesn't necessarily mean increased caching space: Aspects such as the [replication factor](https://opensource.docs.scylladb.com/stable/architecture/architecture-fault-tolerance.html) and [consistency levels](https://opensource.docs.scylladb.com/stable/cql/consistency.html) come into play: If you replicate data across nodes, you can tolerate the failure of nodes and prevent cache misses on frequently accessed data, but this typically halves your effective cache size.

This project began by measuring how many items we could store to each datastore. Throughout our tests, the key was between 4 to 12 bytes (key0 .. keyN) for Memcached, and 12 bytes for ScyllaDB. The value was fixed to 1000 bytes.

#### Memcached

Memcached stored roughly 101M items until eviction started, achieving very high memory efficiency. Out of Memcached’s 114G assigned memory, this is approximately 101G worth of values, without considering the key size and other flags:

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXc4rLxGn-D5lbB8k377973zna4HUI079m_PxYf03tdpYBOQVF-rTZjPOXazl1a37SlC0oWrXGDCqBajtCdesOBgXOmXvChF-McsTNlqfHWlPDSRGXs9oby5p4cTAikHpU7VelYouzOUROzI822nHUOz0CQ?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>Memcached stored 101M items in memory before evictions started</p></figcaption></figure>

The memcached **STATS** output shows how many items and bytes it ended up consuming:

```
STAT bytes 107764057859
STAT curr_items 100978499
```

It is worth noting that each item in memcached introduces a small overhead. To measure it, memcached provides a [sizes](https://github.com/memcached/memcached/blob/master/sizes.c) utility, which output is as follows in our server machine:

```
~/memcached-1.6.25# ./sizes
Slab Stats    64
Thread stats    2352
Global stats    224
Settings    336
Item (no cas)    48
Item (cas)    56
extstore header    12
Libevent thread    488
Connection    496
Response object    1184
Response bundle    32
Response objects per bundle    13
----------------------------------------
libevent thread cumulative    6936
Thread stats cumulative   	 6448
```

For our testing (with CAS disabled), each item introduced an overhead of 48 bytes, which translates to roughly 4.85GB RAM.



#### ScyllaDB

ScyllaDB stored between 60 to 61M items before evictions started to happen. This is no surprise, given that its protocol requires more data to be stored as part of a write (such as the write timestamp since epoch, row liveness, etc). ScyllaDB also persists data to disk as you go, which means that Bloom Filters (and optionally Indexes) need to be stored in memory for later efficient disk lookups.

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXdAR1FAauRr94aYmxr1_ySa8-sXEXXz6T_S27h35PjH6oRFVS6ERWTT4dLghmiSuUiuQVmk98SJYY-T6hx3YpGyw7kbxhg7SE5LS5m9q1tSBB4PyY_NwUWjxw1s6FtliM4nHKPVOK5gnu10EwqAhzrYlmpT?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>Eviction starts under memory pressure while trying to load 61M rows</p></figcaption></figure>

Another difference (and source of overhead) relates to supporting a richer wide-column data model. Note how the number of rows is double the number of partitions stored in the ScyllaDB Cache. This happens because we require storing _range continuity_ information, as we explained in [this article](https://www.scylladb.com/2024/01/08/inside-scylladbs-internal-cache/). However, we found out that we don't need to store dummy rows for continuity information on tables with no clustering keys, and [scylladb/2972](https://github.com/scylladb/scylladb/issues/2972) turns out to be an interesting optimization in that regard.

If you followed closely, then you probably realized we mentioned ScyllaDB stored 60M records, but the provided monitoring snapshot shows evictions happened when we had roughly 52M items within the cache. What happened to the other 8M rows?

ScyllaDB first stores data within memtables and, only during a flush, this data is populated and merged with its cache contents. What happened is that a memtable flush required evicting records from the cache to free memory. However, a memtable is by definition an in-memory structure. Thus, we need to account for the rows within the cache until the last flush prior to eviction, plus the memtable contents.

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXdbqpYVpCHSPBternMrtMrM6kRFH_nOsrwhL7A_z5MwVt2w3zKbgXMd_7N2Z8U6hbXsUPWDGJdIv370qZij6VsZzxCkbLrQFg7Z6CDP3vQjCU9Zle8hyelg5u4BU9syZcHpUHn5eXVX7fjYEA82oZ6nnIQ?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>No evictions at 60M items, 53.3M partitions in Cache, whereas the rest are within Memtables</p></figcaption></figure>

Take this Monitoring snapshot, where no evictions happened. In [this test](https://github.com/fee-mendes/shredders/blob/main/results/scylladb-population), we've populated 60M unique records, whereas in the previous one we populated 61M records. ScyllaDB unfortunately doesn't provide a way to measure the number of partitions within memtables as it is part of the hot write path, but there are ways to estimate that number, such as how we've done here.

ScyllaDB Memory is split in two categories:

* LSA (Log Structured Allocator) – Used by the Cache and Memtables
* Non LSA – Regular memory

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXfXiE8iFWgZ-C_SAE7-l6Yp93DdIDiPNFD6Aou3jcxpjoCbJBDyVRJPHsstuwUWMs2hLuW-1LjNOC86hGHwplX6JZ6tvtvF6EVYWND4B2zI54d0EnMTvDz9TLpIPiBV26iVfhspLF13wBamd-Q214DOG8vu?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>Here we can see that both Cache and Memtables are fully utilizing the server's memory, whereas other components (such as Bloom Filters) consume roughly 2.3GB of RAM.</p></figcaption></figure>



[^1]: Keep in mind that specially during tests maxing out the promised Network Capacity, we noticed [throttling shrinking down the bandwidth down to the instances' baseline capacity](https://twitter.com/dvassallo/status/1120758102985285633).

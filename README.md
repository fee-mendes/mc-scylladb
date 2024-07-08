---
coverY: 0
layout:
  cover:
    visible: false
    size: full
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: false
  outline:
    visible: true
  pagination:
    visible: true
---

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

```bash
STAT bytes 107764057859
STAT curr_items 100978499
```

It is worth noting that each item in memcached introduces a small overhead. To measure it, memcached provides a [sizes](https://github.com/memcached/memcached/blob/master/sizes.c) utility, which output is as follows in our server machine:

```bash
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

### Read-only In-Memory Workload

The _ideal_ workload for a cache (though unrealistic) is one where data can fit in RAM so that reads don't require touching the disk and no evictions or misses happen. Both ScyllaDB and Memcached employ a LRU logic for freeing up memory: When the system runs under pressure, items get evicted from the LRU's tail, which are typically the least active items.

Taking evictions and cache misses out of the picture helps measure and set a performance baseline for both datastores. It places the focus on what matters most for these kinds of workloads: read throughput and request latency.

In this test, we first warmed up both stores with the same payload sizes used during the previous test. Then, we initiated reads against their respective ranges for 30 minutes.

#### Memcached

A great feature of the [memcached protocol](https://github.com/memcached/memcached/blob/master/doc/protocol.txt) is pipelining of requests, which it has supported since its early days. This allows a client to pack multiple requests under a single TCP request, resulting in less network RTTs.

For this test, we've run the mcshredder [_perfrun\_metaget\_pipe_](https://github.com/fee-mendes/shredders/blob/main/suite-performance-lib.lua#L180C10-L180C30) workload with 32 threads, 40 clients and a pipeline depth of 16 items. You can find the (long) output of that run [here](https://github.com/fee-mendes/shredders/blob/main/results/readonly-samegroup-max-nic).&#x20;

Memcached achieved an impressive 3 Million Gets per second, fully maximizing AWS NIC bandwidth (25 Gbps)!

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXfoBB_nencRd5ZQOreDvBAYXwgE_r8TR0o9ZHilpAMBpOeYxZlciNP7nPNiy6uCwzPHAXGHMH4ovzmMGP4CJHcdjjUXFXBRQr2wTp-EVvH8Vrj1jtDkvyNH3PzLSGkyx4i2mMVvtCtGzGhf7Ui0xbos5OHV?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>Memcached kept a steady 3M rps, fully maximizing the NIC throughput</p></figcaption></figure>

We then parsed the test output into our modified version of [perf-parser.pl](https://github.com/fee-mendes/shredders/blob/main/perf-parser.pl), resulting in a histogram showing that the p99.999 of responses completed below 1ms:

```bash
--- subtest basic ---
stat: cmd_get :
 Total Ops: 5503513496
 Rate: 3060908/s
=== timer mg ===
1-10us    0    0.000%
10-99us    343504394    6.238%
100-999us    5163057634    93.762%
1-2ms    11500    0.00021%
2-3ms    144    0.00000%
3-4ms    336    0.00001%
4-5ms    400    0.00001%
11-12ms    32    0.00000%
```

At this point, it becomes clear networking is a bottleneck and memcached likely would've been able to deliver even higher throughputs through a faster link (or without networking at all). For example, dormando mentioned having been able to scale it up past 55 million read ops/sec on [this HackerNews thread](https://news.ycombinator.com/item?id=17179348) for a considerably larger server.

As our main testing focus on performance between distributed systems, we decided not to stress memcached locally, but readers should be aware of the cost optimization opportunities involved: as memcached requires much less CPU for a similar workload, you can switch to smaller instance sizes for a better cost return.

#### ScyllaDB

Contrary to Memcached, ScyllaDB – or more specifically the CQL protocol (subject of our testing) does not introduce a concept of pipelining requests. In that sense, on top of the overheads discussed during the previous test, achieving a similar throughput under a simple key-value workload turns out to be challenging. This happens because – similarly as memcached pipelining is meant to require less client-server round-trips (and thus less Network packets), ScyllaDB clients would require an order of magnitude more requests to achieve the same GET rates seen in memcached, up to a point where we could easily max out the NIC and/or Network max packets per second (pps).

It is worth noting that – while the CQL protocol DOES allow one to read from multiple keys within a single query (via the _**SELECT IN (...)**_ clause), relying on it wouldn't be optimal either, as selecting multiple keys under a single request would have a high probability of spanning multiple replica shards, whereas a single operation would require to be handled by a single coordinator, thus introducing cross CPU traffic and potentially elevating latencies.

At this point, we needed to come up with a better data model for ScyllaDB where a single client operation could result in more "_GET_" requests (or – in ScyllaDB terms, more rows being read per operation). After looking into the problem from a different angle for a few minutes, we've come up with a simple answer for it: Group 16 rows per partition (our memcached pipeline depth) with the help of a clustering key, where each partition hit would result in the server returning us 16 rows.

A similar Memcached workload would simply associate the data from all rows with one key and scale accordingly. However, in doing so, this important difference among both solutions wouldn't stand out, effectively showing Memcached as a more performant solution for single and small key-value lookups.

With a clustering key we could fully maximize ScyllaDB's cache, resulting in a significant improvement in the number of cached rows. We ingested 5M partitions, each with 16 clustering keys, for a total of 80M cached rows.&#x20;

As a result, the number of records within the cache significantly improved (note that the [previous memtable explanation](./#scylladb) still applies here). We've also confirmed the number of total rows matched our expectations prior to our read tests. To do that, we ran an [efficient full table scan](https://www.scylladb.com/2017/03/28/parallel-efficient-full-table-scan-scylla/) which returned 80M rows.

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXfS6TI0q5lhuHHLlbdSZDsY4vqNsYbeaXaKCswAiIlxhvQM2RIuLb7Ts_iUzAeVQ9v8PwI4ecPskqUdQxuWIDvwxkmSUWBmCvWMAdmKQ0e_MkxOk1H7Gbu-MIhxI4QXuunRMkzCU55Ma9qjJtY5VzlqsttD?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>ScyllaDB Cache utilization after initial data ingestion</p></figcaption></figure>

With these adjustments, our loaders ran a total of 187K read ops/second over 30 minutes. Each operation resulted in 16 rows getting retrieved.

Similarly to memcached, ScyllaDB also maximized the NICs throughput. It served roughly 3M rows/second solely from in-memory data:

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXduI9-hkM1GIckgPl6jz9ib69tykKUr-D7uYfB6HSnyZH1xLDeIMsYcZkNVlhVTtsODTa_63mPyyZBPNpnDOSPTtdSoLzZtSUcXIXchNRDSWMdWzFsVSSMa4vA8PsmNtyOHpPdz9UyT4_25jI2FvDVRzy4?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>ScyllaDB Server Network Traffic as reported by node_exporter</p></figcaption></figure>

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXc1J9fgm8YJiMvpMflZwPg_DL08hXT1maPxC6YKiwLXhQZcLWDHZgQB2XlgArJxXYoRfLWDudpf6d1ufnQTzqMt303TuCuSaPQC31YQ_osGh6PM9EADQi5GBraqIebPVScUZo185o1y4GB0kTpOczamk4Aj?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>Number of read operations (left) and rows being hit (right) from cache during the exercise</p></figcaption></figure>

ScyllaDB exposes server-side latency information, which is useful for analyzing latency without the network. During the test, ScyllaDB's server-side p99 latency remained within 1ms bounds:

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXcddRmSJ-atsKgIgmGKchJU3ljenY--8l_CcmEf8K6ZbKfKjqwoeu3rIagICzxQc8IuGfj94iQJEyRwVjo96X9wWzUU6u2i8EdrnzHlzNj7UAJ5E47elAAb_Tw6xE4V9IRImRfU1hYofk3RtQCyb9u2sYI?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>Latency and Network traffic from ScyllaDB matching the adjustments done</p></figcaption></figure>

The client-side percentiles are, unsurprisingly, higher than the server-side latency. The following graph demonstrates the P99 spanning a period of 5 minutes from clients' perspective:

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXeyHq6b7CsGtvyiAnhlm3xQzHOgoEUe2U1cGA7AP3lNtFiSva3RRrtUXjAueTqQnX0FkHRx_Fn1qF37jZztv3_AgfGQd2ttAJPtC8Atgod2Zpu7Xt9e040M2L9VUdVSlDUTPsCWMLvr-V3VGY8Izv2iVwg?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>cassandra-stress P99 latency histogram</p></figcaption></figure>

### Adding Disks into the Picture

Measuring flash storage performance does introduce its own set of challenges, which makes it almost impossible to fully characterize a given workload realistically. During our planning phase we did discuss what would have been a proper "ideal workload" and – the hard truth is that such a thing doesn't exist.

That is, the ideal use case for a cache is one where frequently accessed and recent items live in RAM. As demonstrated earlier, in-memory items have an unlimited fetch ceiling and maximize throughput well beyond what modern Cloud NICs can offer. Conversely, if the item isn't frequently accessed then it gets offloaded to disks, as the likelihood of having it being hit again is lower when compared to existing items in memory.

Disks have a much lower fetch rate compared (and as opposed to) RAM. If we were to stress disks to their limits we would be finding ourselves testing against what a proper cache workload is meant for. These characteristics alone, took us to an impasse, with no easy way around it.

Given these constraints, we decided to measure the most pessimistic situation: Compare both solutions serving data (mostly) from block storage, knowing that:

* The likelihood of realistic workloads doing this is somewhere close to zero
* Users should expect numbers in between the previous optimistic cache workload and the pessimistic disk-bound workload in practice.

Prior to any tests, we measured our disks performance using [fio](https://github.com/axboe/fio) and recorded its [output](https://github.com/fee-mendes/shredders/blob/main/results/fio). We used the XFS filesystem with its default settings. We were particularly interested in the randread test, showing that the disk delivered up to 340K IOPS with a bandwidth of 1.3G/s for a block size of 4kB, as many small I/O operations match the behavior of how datastores dispatch requests to underlying storage.

Next, we moved on to load data and started a storage-bound workload against both solutions.

#### Memcached Extstore

Offloading in-memory cache space to flash storage is not necessarily a new thing. Memcached [introduced](https://github.com/memcached/memcached/wiki/ReleaseNotes154) support for Extstore in 2017, and dormando provided a thorough explanation around [the case for NVMe](https://memcached.org/blog/nvm-caching/) in his later 2018 article, providing insights on the tradeoffs around performance and costs still relevant to date.

The [Extstore](https://github.com/memcached/memcached/wiki/Extstore) wiki page provides extensive detail into the solution’s inner workings. At a high-level, it allows memcached to keep its hash table and keys in memory, but store values onto external storage. Keep in mind that using Extstore is currently incompatible with [warm restarts](https://github.com/memcached/memcached/wiki/WarmRestart#future-work), thus a process restart always invalidates all data stored within the filesystem.

The good thing about Extstore's approach is that it is very simple to reason about. Items are written mainly to RAM and stored in a particular [Slab class](https://github.com/memcached/memcached/wiki/UserInternals#how-memory-gets-allocated-for-items). [During memory pressure, rather than evicting records, the item values are asynchronously flushed to storage and its key and other necessary structures are reallocated to a different slab class.](#user-content-fn-2)[^2] On top of the per item overhead, memcached also needs an additional 12 bytes per item containing a pointer to the flash location where the item got stored for efficient retrieval later.

Given the in-memory item overhead, it is important to note that even with Extstore there will still be a limit to the number of items you will be able to store under a single memcached instance. During our tests, we populated memcached with 1.25B items with a value size of 1KB and a keysize of up to 14 bytes:

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXeiah9qsaBsgiNF0bVcdY1JDGNoYkpG0m9x6e-5LG3-fLkmPlO524xd-sgeFiiEFFu72RiSnDUdlCYbQvSknFz_7ERArIOc7qeQve3LJqR1jb2uqpLiwev2lSzUEhuueE9HUHtO99jJfLEzEpPyR8WyVU8b?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>Evictions started as soon as we hit approximately 1.25B items, despite free disk space</p></figcaption></figure>

With Extstore, we stored around 11X the number of items compared to the previous in-memory workload until evictions started to kick in (as shown in the right hand panel). Even though 11X is an already impressive number, the total data stored on flash was only 1.25TB out of the total 3.5TB provided by the AWS instance.

Obviously larger values will incur a higher storage consumption and, with fewer keys being needed to accomplish such a task, result in better memory utilization. In a sense, we find important to call out this detail (and memcached documentation even mentions that "the smaller the items stored on flash, the more likely you are to saturate the flash device before your network device"), as you probably want to save some RAM space for serving hot items, without having to incur the penalty of seeing Extstore IO threads frequently hitting your disks.

Speaking of IO threads, Extstore relies on buffered IO (using `pread()`, `pwrite()` syscalls) to dispatch requests to its underlying backing store. The `ext_threads` parameter is used to control the number of threads available for Extstore and finding the optimal thread count is up to the user: A small number can leave the disk underutilized, whereas a too high number can introduce thread contention. Support for `O_DIRECT` and asynchronous I/O are planned within Extstore's roadmap to address some of the shortcomings involved with relying solely on the Linux page cache.

We slightly modified memcached CLI arguments for our performance tests. We introduced extstore relevant parameters and no longer performed any CPU pinning for network IRQs, provided that the previous fio numbers had already shown that a disk-bound traffic would be unable to fully maximize our available network bandwidth. The command line used was:

```bash
/usr/local/memcached/bin/memcached -v -A -r -m 114100 \
 -c 4096 --threads 16 -u scylla -C -o ext_path=/mnt/file:3400G \
 -o ext_threads=<thread_count>,ext_wbuf_size=32
```

On top of these settings, we also ensured that our backing NVMe store had no defined IO scheduler (none in sysfs), _and_ [_nomerges_](https://www.kernel.org/doc/Documentation/block/queue-sysfs.txt) was set to 2.

**Small Payloads**

We started by measuring Extstore's performance with an item size of 1KB (excluding the key size). During those tests, we wanted to keep a fine balance between available item memory and storage utilization. Considering a per-item overhead of 48 bytes, plus 12 bytes for the flash pointer, and a key size ranging from 5 to 14 bytes, we stored 700M items for approximately \~52GB of RAM utilization, and approximately 700GB of raw values.

Our first test involved loading the data using 32 memcached IO-Threads, showing that the disks were underutilized:

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXfGrfPakWYT9U3Ra1byG60uf8f5RmSSIshBQc60Juh8JIz-EgcC4FmXaf9e1kSK6y6JyfnInqFSC-RJaGiTmWD4TUzbyDPa09HJ7wc4C5FVErpv2jO4OKs-QVXODd4yom73qgHq5n3O2k9qy-QvdBq70Z_v?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>iotop: I/O utilization with 32 IO-threads</p></figcaption></figure>

Even though all IO threads were serving reads, we weren't able to increase throughput further without observing a latency increase on the client-side (matching an increase on memcached's `extstore_io_queue` metric) . Next, we doubled the number of IO Threads to 64, resulting in higher throughput:

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXeaLZTQNwjQGl8Gu3-qFR_Ie4HcvLFfh4HTkLf5y6vb7d4O7TOz_HTlvyGVkRQLX5Pngi6-vHhvtEbuLNOXwBw6BmJyP87PdLUyHk89aSEIuJlAhvU0_f2TxsY-9RxWOsPgul5RDh8I5MiG0IxmRz6Yu6yK?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>iotop: I/O utilization under 64 IO-threads</p></figcaption></figure>

The table below summarizes the numbers we've observed while reading with a pipeline depth of 16, as well as with no pipelining:

| Test Type                                                                                                        | IO Threads | GET Rate | P99 Latency |
| ---------------------------------------------------------------------------------------------------------------- | ---------- | -------- | ----------- |
| [perfrun\_metaget\_pipe](https://github.com/fee-mendes/shredders/blob/main/results/readonly-disk-1kb-32threads)  | 32         | 188K/s   | 4\~5 ms     |
| [perfrun\_metaget](https://github.com/fee-mendes/shredders/blob/main/results/readonly-disk-1kb-nopipe)           | 32         | 182K/s   | <1ms        |
| [perfrun\_metaget\_pipe](https://github.com/fee-mendes/shredders/blob/main/results/readonly-disk-1kb-64threads)  | 64         | 261K/s   | 5\~6 ms     |
| [perfrun\_metaget](https://github.com/fee-mendes/shredders/blob/main/results/readonly-disk-1kb-64threads-nopipe) | 64         | 256K/s   | 1\~2ms      |

**Larger Payloads**

We decided to store 8KB values and target close to 75% disk utilization for our large item tests. Next, we started mcshredder [disk test](https://github.com/fee-mendes/shredders/blob/main/disks.lua) to start reading from those items.

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXdYrSPIkG0TbBsUV7okcfHLcoMXVK9mAoQqOr3fd8sjZDj_wkVY-TtCB1DkUrDgFPWX1Ehc2efNbY4WIAxcEUKMg1tTepeiAFYWnhWp_D9XIlN8f_7yW4Nzg6QigaIgPuHCjYiD7hGRgl_AioLyN1IYMvzf?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>Larger payloads shine on Extstore and can achieve a higher disk utilization</p></figcaption></figure>

Unsurprisingly, in addition to less overhead and achieving higher disk utilization, an added benefit of storing larger objects in storage is the fact that it can achieve even higher ratios of extended caching space. If we were not using Extstore, we would need somewhere close to 25X more RAM to store the same \~350M 8KB objects (2.8TB total). There are clear cost savings achieved by leveraging Extstore within your existing Memcached infrastructure.

As expected, the results demonstrate that latency was higher when reading primarily from disks than memory. The following table summarizes the results found under different tests while using different thread counts for Extstore:

| Test Type                                                                                                    | IO Threads | GET Rate | P99 Latency |
| ------------------------------------------------------------------------------------------------------------ | ---------- | -------- | ----------- |
| [perfrun\_metaget\_pipe](https://github.com/fee-mendes/shredders/blob/main/results/readonly-disk)            | 16         | 92K/s    | 5\~6 ms     |
| [perfrun\_metaget](https://github.com/fee-mendes/shredders/blob/main/results/readonly-disk-nopipe)           | 16         | 90K/s    | <1ms        |
| [perfrun\_metaget\_pipe](https://github.com/fee-mendes/shredders/blob/main/results/readonly-disk-32threads)  | 32         | 110K/s   | 3\~4 ms     |
| [perfrun\_metaget](https://github.com/fee-mendes/shredders/blob/main/results/readonly-disk-nopipe-32threads) | 32         | 105K/s   | <1ms        |

The overall disk bandwidth achieved ranged between 1GB/sec (16 IO threads) to 1.2GB/sec (32 IO threads). We noticed that throughput didn't scale linearly along with the thread count. iotop shows that a single Extstore thread achieved \~66MB/s for 16 IO Threads (dark background), whereas with 32 IO threads (white background) the workload didn't dispatch I/O to all available threads, somewhat a sign that disks are close to saturation.

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXfAXE9s7f2l3z-OBlyicDMH7bbuLsX1d8G0ZWvPOTTZrPySOV5X5y8l1YsLJwSRSzTWluLlahN7GLK-sVMUYCZwOm1P5EzchUBT9XODBZDUt7acHFXsb_nj4RAJ72fU2tp7QMm9JvUqThNZNV3YFXDYQMZQ?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>iotop: I/O utilization under 16 IO Threads, payload of 8KB</p></figcaption></figure>

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXebSJt5fLAysOy3-D5WqvKDXm3Up3SXGSMqoilPt5Fv_5OWGr6MFnK9gS9B0XC9yl7fike0uPlcyNxUW44mDWFyRrnDzq2dt0Lv46F_1VbC25sfvYbOU6BU2lo-w69HvWTCWyY6mhQvV8ik8XMK7u2-ZK7P?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>iotop: I/O utilization under 32 IO Threads, payload of 8KB. Note some threads are mostly idle.</p></figcaption></figure>

A final remark is the fact that workloads without pipelining observed lower latencies, and completed most requests under sub-millisecond response times. In fact, Memcached exposes an `extstore_io_queue` metric which helps us understand the different latency numbers: The I/O queue was considerably higher when using pipelining (about 10X more) as opposed to individual GETs, which somehow make sense, as multikey batches require more I/O to complete on a per request basis.

#### ScyllaDB

As opposed to memcached, ScyllaDB is a full-blown database. This entails – among other things, employing crash recovery mechanisms (the commitlog), supporting different compaction algorithms for different workloads, compression for saving storage space, and storing SSTable components in RAM (such as [Indexes](https://opensource.docs.scylladb.com/stable/architecture/sstable/sstable3/sstables-3-index.html), [Summaries](https://opensource.docs.scylladb.com/stable/architecture/sstable/sstable3/sstables-3-summary.html) and [Bloom Filters](https://enterprise.docs.scylladb.com/enterprise/architecture/sstable/sstable3/sstables-3-data-file-format.html)) to minimize on I/O operations.&#x20;

More recently, ScyllaDB also introduced support for [SSTable index caching](https://youtu.be/nl4pABC4gyQ?si=bvIK4t70On1Tb1wV\&t=1711) in order to further minimize walking through index files on disk. Prior to this feature, all read requests not present in ScyllaDB's row cache would require issuing disk I/O to lookup the position in the Index file where the corresponding key is located. Therefore, ScyllaDB follows a very different approach than Extstore's key-cache implementation and, although ScyllaDB doesn't implement a key-based cache, discussions around the topic do exist ([#194](https://github.com/scylladb/scylladb/issues/194), [#363](https://github.com/scylladb/scylladb/issues/363)).

Beyond these optimizations there is also the complexity of supporting complex data models, access patterns, replication and background operations (such as streaming, repairs and compaction), while – at the same time, guaranteeing that a fair share of resources are given to tasks competing for I/O, without an impact to the end users' latency.

It goes without saying that a database like ScyllaDB requires fine-grained control over every I/O operation, beyond what the Buffered I/O can offer, and we explained the reasons [why we chose Asynchronous Direct I/O for ScyllaDB](https://www.scylladb.com/2017/10/05/io-access-methods-scylla/) a few years back.

One of the challenges involved with AIO/DIO (and as within the previous article) is that its benefits come with a significant cost: complexity. To overcome this, ScyllaDB uses the userspace [Seastar I/O scheduler](https://www.scylladb.com/2022/08/03/implementing-a-new-io-scheduler-algorithm-for-mixed-read-write-workloads/) to prioritize and guarantee quality of service among the various I/O classes that the database has to serve requests for.

Although the implementation is very complex on its own (and we [improved the](https://www.scylladb.com/2016/04/14/io-scheduler-1/) [I/O scheduler](https://www.scylladb.com/2018/04/19/scylla-i-o-scheduler-3/) [generations](https://www.scylladb.com/2021/04/06/scyllas-new-io-scheduler/) [over the years](https://www.scylladb.com/2022/08/03/implementing-a-new-io-scheduler-algorithm-for-mixed-read-write-workloads/)), a notable benefit for users is that getting started with it is particularly simple. During the database setup process, seastar's [iotune](https://github.com/scylladb/seastar/tree/master/apps/iotune) will benchmark its underlying disks and record the disk bandwidth and IOPS for both reads and writes. This process not only helps the I/O Scheduler to make informed decisions and keep the disk concurrency high, but it also prevents the need for fine-tuning disk-related settings, as we did for Extstore threads.

The following output demonstrates the contents of `io_properties.yaml`, used by the I/O scheduler once the database starts. As we can see, the numbers aren't very different from the ones recorded within the previous fio outputs.

```yaml
disks:
  - mountpoint: /var/lib/scylla
	read_iops: 313289
	read_bandwidth: 3116245504
	write_iops: 91691
	write_bandwidth: 2287052288
```

Compared to memcached's method of storing in RAM the relevant pointers to storage for each key, ScyllaDB reads follow a totally different approach:

1. The database first selects a list of candidate SSTables likely to contain the data via its Bloom Filters.&#x20;
2. Next, each SSTable summary file is looked up to estimate the position where the specific key being looked after exists.
3. The estimated positions are then searched through in the actual Index file
4. Once the key is found, a read is issued to the Data file to retrieve the actual data being looked after.

Steps (1) and (2) happen in memory, and step (3) may happen in memory or not, depending whether the Index in question is cached. In a sense, memcached approach favors fast and efficient data retrieval, whereas ScyllaDB favors the ability to store and query through large data sets ranging from hundreds of millions to billions of items.

Given this approach, ScyllaDB isn't configured by default to heavily scan small partitions from disk. That is, in a key-value workload, every partition read would walk through the 4 steps above, inevitably adding a level of overhead. Conversely, wide-rows benefit from the default settings as the number of disk seeks to the Index file is often negligible compared to the amount of sequential reads done to the actual data file.

ScyllaDB does provide optimization options suitable for key-value workloads. In particular, one possible optimization is to increase the cardinality of its Summary files and therefore enhance the precision when looking up the Index file. This task can be accomplished via the sstable\_summary\_ratio configuration parameter, with the tradeoff being an increased size of the summary files stored in RAM. Should you want to read more about this setting, check out [how Pagoda optimized their disk performance with ScyllaDB](https://medium.com/agoda-engineering/exploring-scylla-disk-performance-and-optimizations-at-agoda-65a6dcdd6fe7). Another optimization would be to adjust the index\_cache\_fraction option to minimize Index scans on disk even further.&#x20;

It is worth keeping an eye on ScyllaDB's Non-LSA memory consumption prior to making such a change, as increasing its footprint effectively reduces the total row-cache size, which may be particularly important for storing more items within the database cache.

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXcHip7LCAXjXRpjVN0CcKYFlT2xOvku8r_N4ISH7p3XtuDT1kKTIo5GUJWY36w9iOCkV2qZQE3YNOAgKcGa0j_GU4WJP9xyDfjS0vRc1AtpYxUoQY1fpgtVg69oTDldI3atmNEzEGDwDDoEE2FwG97-RxYG?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>Non-LSA Memory</p></figcaption></figure>

The last overhead relates to the commitlog, a WAL sitting in the hot write path by default. Although ScyllaDB does allow users to disable the commitlog (via the `--enable-commitlog` option), we haven't explicitly set that option, meaning that every write also incurred additional I/O before being acknowledged back to the client.

**Small Payloads**

Following memcached tests, we populated ScyllaDB with 700M items of 1KB. The schema used was a plain simple key-value store, where a single partition holds a single row. On top of these changes, the ScyllaDB `index_cache_fraction` settings also got increased to 0.5 (from its default 0.2) to mimic (though not quite) a similar hash table allocation as observed during the memcached tests.

The [results](https://github.com/fee-mendes/shredders/blob/main/results/scylladb-disk-1kb-read-results) show ScyllaDB achieved an impressive rate of over 19K reads/second per shard with server-side P99 tail-latencies of 2 ms:

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXet33U2fkIa4aKYajx7guGWbUlCIACtDKhVhno9FBdoOA858j391S7Bk0kgN5KL5TXXwIV8IDCGXCDSGeD_ePJJd-8ylfYBdka8FFEoeyHHx92_OahEofzWkKapfEg_SHUB4YGtEtB4xzUAXfnaI3lmXoTL?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>Left: Per core (shard) throughput. Right: P99 Latency</p></figcaption></figure>

On the disk I/O level, note how ScyllaDB required much less bandwidth (but got close to maximizing disk read IOPS) to achieve its throughput, thanks to its AIO/DIO implementation:

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXcyRITR2t44DBjxu9Fj_9LSAnqlo3SjJ5_o9MccR9aiSzsgQ1ogpGMI2t8gPKttkcQaiugsbmgCFO3hRYtImVYiRvtie9u-XoIBjnCJqulmuJaoGVO8HoCupoVn4xJ73tmx1iLI86KMvMe2A34iwoLO20I?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>ScyllaDB I/O scheduler bandwidth and IOPS</p></figcaption></figure>

Finally, the client-side reported a throughput rate of 268K ops/s, under a collective P99 latency of 2.4ms:

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXeQEVVg9Ak6tVoqvRP_9pr3SFDxxL5n96ZjGGFHVak58wZ_A68IaVFHGiMMVQmOPYevgixW7BRiCLwVFOP3dY__6eWctesx0xbZGlJrXiNvDWAhHWnlaDVA9G-hJ-Xl3y_rPTm_rvQUg-5_PLzbuBj0RYvc?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>Client-side latency histogram</p></figcaption></figure>

**Larger Payloads**

During the ingestion phase we populated ScyllaDB with the same 350M records of 8KB in size as we've done for Extstore, bringing the total storage utilization to about 81%, around 6% more than Extstore's consumption.

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXccjWehjzCa0VimolCTChMoUJ_SaNwdBwnJGbHtdAe1gHDxTxBY-e4iLxmtIgaIi_aOaqtCWRPFsCYLaw4TlNZ_25s1Xe86qxvKFwqOp-kcjQ7GLcOP80ugqAuXjk9ZYU3tr-NWz4fMgybYsggox-jAGFy1?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>ScyllaDB storage utilization after ingesting 350M ~8KB records </p></figcaption></figure>

Our next and final step was, of course, to read from this data. Before doing so, we applied the following optimization settings for reading from this particular key-value workload:

* `cache_index_pages: true`
* `--io-latency-goal-ms 1`

On top of these settings we also changed the compaction strategy to [Leveled Compaction](https://opensource.docs.scylladb.com/stable/architecture/compaction/compaction-strategies.html#leveled-compaction-strategy-lcs), a great fit for read-mostly workloads, and turned off [SSTable compression](https://opensource.docs.scylladb.com/stable/cql/ddl.html#compression-options), primarily as most of our data was stored using blobs.

The [results](https://github.com/fee-mendes/shredders/blob/main/results/scylladb-disk-read-results) summary shows that ScyllaDB delivered close to 160K reads/sec with a P99 latency below 2ms entirely from disks. The following monitoring snapshot, shows that no keys within ScyllaDB's row cache got touched during the run (except for client reads during startup):

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXcUg_zkKrfgVRxHJnui57HpFxIBfQe31kFmUA1aBZqjYOMcUSdGqnbM8coGZh8gthKp0qyCAQbWrdO0Vk7hqoZHUEMP5b3GTl1jzMi4FlxDUMA-dGCHabg0HWWkxorjPjk0IEb8XdcLnSg3Br-6MljaHu8?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>Except during the initial connection stage, all traffic is being directed to disk - Note how most traffic results in a Cache Miss</p></figcaption></figure>

The following graph demonstrates the P99 spanning a period of 5 minutes from clients' perspective:

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXeXoc7sXKeE5hiXcK2Z-VQ80EvplO5k9-Jl4xQDANlZY7OxCfsSlHREAqegtWidGXZAr4yx3DXlUPrnt1Vra7W21n_FhSjRBWY_qlFnCrmNfHcK_C-CfeZijfi5_tq8FTP6jc_JOp01wxjH1WudNQPdqmM?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>cassandra-stress P99 latency histogram</p></figcaption></figure>

Despite the increased rate, the number of IOPS and bandwidth achieved was very similar to memcached's numbers. Note how starvation time increases over time, yet another indication that the disks cannot afford timely dispatching for some requests (thus increasing the requests' latency):

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXcuPlnZjlLFQk01gM6kDh-9rY0wFTi6PGqlWXkrfLLNLZi2KYDxipCpyFRarxDIH-toP8T9f8ORZnmSjstZgyg6ViQUu6HaKrhvme-y5V1nULhJZePgxyLRuDgF7-HnzVeJKB3-G-Nl-22ocJDjy6bjCTVg?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>ScyllaDB I/O Scheduler Metrics</p></figcaption></figure>

Finally, as a comparison to Extstore results, note how ScyllaDB achieved higher (although similar) bandwidth than memcached's 32 IO threads:

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXdA9clz_diii-jHv8NME2J5m4y2Zw0KOuFsTgm0EEluy_7dAyJTo9_wEGOQADYhpbUiZZMRAQRma9VlookxywZ3spkPMQ0G5Em1vqJ5jPzs1qJrO6O8sK735vGrUsalhM37TVse4DaHI64F8v0C29GWscgm?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>iotop: ScyllaDB I/O</p></figcaption></figure>

### Overwrite Workload

Following our previous Disk results, we then compared both solutions in a read-mostly workload targeting the same throughput (250K ops/sec). The workload in question is a slight modification of memcached's ['basic' test for Extstore](https://github.com/memcached/shredders/blob/main/suite-extstore-tests.lua#L36), with 10% random overwrites. It is considered a "semi-worst case scenario". This does make sense, as Cache workloads naturally shine upon higher read ratios.

#### Memcached

Memcached achieved a rate of slightly under 249K during the test. Although the write rates remained steady during the duration of the test, we observed that reads slightly fluctuated [throughout the run](https://github.com/fee-mendes/shredders/blob/main/results/overwrite-disk-1kb-64threads):&#x20;

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXfsuq-4V0SkE0r2MstisP3C10JUa44Lble0uE64gY_qon9x9r3cf5YCgzSozshzbF8eTGOdWZoiw8hjevPYKRCz4Q7yfS3jQaor8563MYWj1zHccI4EvqEx0QYHvQUYAzQ8oDRzvMtnN0p-NZ7bx1xu6V4C?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>Memcached: Read-mostly workload metrics</p></figcaption></figure>

We also observed slightly high `extstore_io_queue` metrics despite the lowered read ratios, but latencies were still kept low. These results are summarized below:

| Operation | IO Threads | Rate    | P99 Latency |
| --------- | ---------- | ------- | ----------- |
| cmd\_get  | 64         | 224K/s  | 1\~2 ms     |
| cmd\_set  | 64         | 24.8K/s | <1ms        |

#### ScyllaDB

The ScyllaDB test was run using 2 loaders, each with half of the target rate. Even though ScyllaDB achieved a slightly higher throughput (259.5K), the write latencies were kept low throughout the run and the read latencies were higher (similarly as with memcached):

<figure><img src="https://lh7-us.googleusercontent.com/docsz/AD_4nXeGS9GgLrw3UYGcCFdnXoRPj3A6Lk3my-ura3kviAYacw9x_ey6OYHtEQT5Rwdo5Phw3tnC1qRpkFplmzE6RF2R60w1p-_giOQ8RGR7MbnD2UHxsjPAr4r9vROJ762_LOxVqXytMrGPf76eGzWdv7EgcNo?key=9Ux2m8sYTFJj8yEnmFCj9A" alt=""><figcaption><p>ScyllaDB: Read-mostly workload metrics</p></figcaption></figure>

The table below summarizes the client-side run results across the two loaders:

| Loader                                                                                                          | Rate     | Write P99 | Read P99 |
| --------------------------------------------------------------------------------------------------------------- | -------- | --------- | -------- |
| [loader1](https://github.com/fee-mendes/shredders/blob/main/results/scylladb\_overwrite\_disk\_loader1-results) | 124.9K/s | 1.4ms     | 2.6 ms   |
| [loader2](https://github.com/fee-mendes/shredders/blob/main/results/scylladb\_overwrite\_disk\_loader2-results) | 124.6K/s | 1.3ms     | 2.6 ms   |

## Wrapping Up

Both memcached and ScyllaDB managed to maximize the underlying hardware utilization across all tests and keep latencies predictably low. So which one should you pick? The real answer: It depends.

If your existing workload can accommodate a simple key-value model and it benefits from pipelining, then memcached should be more suitable to your needs. On the other hand, if the workload requires support for complex data models, then ScyllaDB is likely a better fit for your needs.

Another reason for sticking with Memcached: it easily delivers traffic far beyond what a NIC can sustain. In fact, in [this HackerNews thread](https://news.ycombinator.com/item?id=17179348), dormando mentioned having been able to scale it up past 55 million read ops/sec for a considerably larger server. Given that, one could make use of smaller and/or cheaper instance types to sustain a similar workload, provided the available memory and disk footprint suffice your workload needs.

A different angle to look at is the data set size. Even though Extstore provides great cost savings by allowing you to store items beyond RAM, there's a limit to how many keys can fit per GB of memory. Workloads with very small items should observe smaller gains compared to those with larger items. That’s not the case with ScyllaDB, which allows you to store billions of items irrespective of their sizes.

It’s also important to consider whether data persistence is required. If it is, then running ScyllaDB as a replicated distributed cache provides you greater resilience and non-stop operations, with the tradeoff being (and as memcached [correctly states](https://github.com/memcached/memcached/wiki/ProgrammingFAQ#how-do-you-handle-replication)) that replication halves your effective cache size. Unfortunately Extstore doesn't support warm restarts and thus the failure or maintenance of a single node is prone to elevating your cache miss ratios. Whether this is acceptable or not depends on your application semantics: If a cache miss corresponds to a round-trip to the database, then the end-to-end latency will be momentarily higher.

With regards to consistent hashing, memcached clients are responsible for distributing keys across your distributed servers. This may introduce some hiccups, as different client configurations will cause keys to be assigned differently, and some implementations may not be compatible with each other. These details are outlined in Memcached [ConfiguringClient wiki](https://github.com/memcached/memcached/wiki/ConfiguringClient). ScyllaDB takes a different approach, where consistent hashing is done on the server level and propagated to clients when connection is first established, thus ensuring that all connected clients always observe the same topology as you scale.

So who won (or who lost)? Well… This does not have to be a competition, nor an exhaustive list outlining every single consideration for each solution. Both ScyllaDB and memcached use different approaches to efficiently utilize the underlying infrastructure. When configured correctly, both of them have the potential to provide great cost savings.

We were pleased to see ScyllaDB matching the numbers of the industry-recognized Memcached and – of course, we had no expectations of seeing our database being "faster". In fact, as we approach microsecond latencies at scale, the definition of faster or not becomes very subjective. :-)

## Q\&A

**Question:** Why were tests run against a single ScyllaDB node?

* Answer: The typical ScyllaDB deployment indeed starts with a 3-node cluster, typically spread out across different Cloud availability zones. In this testing, we wanted to compare ScyllaDB against memcached in an apple to apples comparison. Although it is perfectly possible for memcached to also be run in a distributed fashion, we decided to keep the testing simple.



**Question:** Why did you disable CAS on memcached?

* Answer: To save on per-item overhead. Refer to [memcached (How Much Memory Will an Item Use)](https://github.com/memcached/memcached/wiki/UserInternals#how-much-memory-will-an-item-use) page.

\
**Question:** Why didn't you introduce elevated write rates?

* Answer: Caches are a great fit for Read-intensive workloads. An elevated write ratio introduces eviction (as data eventually can no longer fit within the cache), and may end up introducing an anti-pattern known as Cache Thrashing, not to mention the risk of evicting important cached items due to overcaching – effectively doing more harm than good.



**Question:** Why haven't you tested \<add particular workload> or \<add specific configuration>?

* Answer: It is almost impossible to test ALL particularities. This work has been peer reviewed, and we believe the presented scenarios are close enough and relevant to a majority of users. Yes, there will exist conditions where one solution will shine more than others. We showed you how it's done, feel free to compare your particular solution/settings there and share your results.



**Question:** Why memcached and not Redis, ElastiCache, Memorystore, etc?

* Answer: Fully Open Source without vendor specific code, everything else happened naturally. Plus, memcached is really simple to get started with.

\
**Question:** Why not use YCSB, nosqlbench, memtier or \<add your favorite> instead?

* Answer: Some of these tools are outdated (YCSB is a prime example of this) and/or may not follow each solution's best practices. We'll be happy to use your tool next time should you be able to prove it can attain similar or better results as we did with ours..

[^1]: Keep in mind that specially during tests maxing out the promised Network Capacity, we noticed [throttling shrinking down the bandwidth down to the instances' baseline capacity](https://twitter.com/dvassallo/status/1120758102985285633).

[^2]: In reality, memcached documentation states that "If you're writing new data to memcached faster than it can flush to flash, it will evict from the LRU tail rather than wait for the flash storage to flush. This is to prevent many failure scenarios as flash drives are prone to momentary hangs". This shouldn't generally be a concern, as cache workloads are suitable for ephemeral data.

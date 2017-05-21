# PQC (ProxySQL Query Cache) Internals

## References

> PQC = ProxySQL Query Cache
> MQC = MySQL Query Cache


## Basics

Follow up the discussion in its orinal thread at [Issue 890](https://github.com/sysown/proxysql/issues/890).

Note that the nature of ProxySQL's Query Cache is very different than MySQL's Query Cache.

The second note is that the micro-benchmark I ran in the past never showed ProxySQL's Query Cache as a bottleneck. Maybe it is time to run new micro-benchmark and publish result!


## Implementation Details:


- PQC has already multiple caches, currently [hardcoded](https://github.com/sysown/proxysql/blob/master/include/query_cache.hpp#L8) to 32 caches. 32 is a random large number and not configurable as proposed in [#64](https://github.com/sysown/proxysql/issues/64)
- In MySQL Server it is possible that hundreds (or thousands) of threads (one-thread-per-connection) try to access MQC: this creates a lot of contention! In ProxySQL, the maximum number of threads that can try to access PQC is mysql-threads only (+1 , more details below).
- MQC invalidates resultset on execution of DML against the specified tablename: this complex mechanism, often the real cause of MQC being a bottleneck, doesn't exist in PQC, at all.
- PQC invalidates resultsets on TTL, exactly as a KV storage like memcached does
- MQC performs the search based on the query, and the invalidation based on a list of tables; PQC performs the search only based on a key. PQC is a KV storage!
- When a thread accesses MQC, it does all the required housekeeping: invalidation, memory management, etc.
- When a thread accesses PQC, the only housekeeping it can potentially perform is to mark the found resultset as expired and remove it from the hash table. It doesn't even bother to deallocate the resultset from memory!
- All the housekeeping is performed by a background thread, the [Query Cache Purge Thread](https://github.com/sysown/proxysql/blob/master/doc/query_cache.md#purging-thread)

For all the reasons listed above, the computation performed by a MySQL Thread while accessing PQC is minimal. 
Because the computation performed inside PQC is extremely small compared to the computation performed outside PQC (for example, 
Query Processor is one of the most CPU intensive module), is it extremely unlikely that multiple MySQL Threads are trying to 
access the same PQC's hash/partition at the same time.

## Current caveats

Using ProxySQL's cache (PQC), via a TTL, for the length of the time of the TTL, data could potentially be returned to the client that is stale.

Currently TTL, is the only way to invalidate entries in the PQC. 

## Possible improvements


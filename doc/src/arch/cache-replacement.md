# Cache Replacement

Object and Chunks are evicted independently.

The rule is: if a chunk is accessed, the object it belongs to is accessed too.
This way It's very likely an object won't be evicted if there are still chunk accesses to
it.

- It is possible that a chunk is evicted while the other chunks still resides in CacheServer.

- It is possible that an object resides in CacheServer but all its chunks are
    evicted.

- It is possible that an object is evicted but there are still orphan chunks.
  These chunks will be evicted finally since there won't be any access to them.

The cache replacement policy is a modified and simplified version of `LIRS`.



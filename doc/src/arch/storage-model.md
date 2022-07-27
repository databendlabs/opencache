# Storage model

CacheServer stores object(a byte stream identified by a unique utf-8 string key) in chunks.
Chunks belonging to a same object have equals sizes.

- When reading an object, the first and last chunk may be partially returned if
    the specified range does not align to chunk size.

- When writing an object, the first and last nonaligned bytes will be discarded.
    Because opencache stores object content in fixed size chunks.

    Optionally, the chunk size can be specified when an object is added(the
    first write) and won't be changed.

## Chunk

A `Chunk` is barely a byte slice.

Chunk size can be specified by an application(when the first time writing an
object) or decided by opencache server.  The default chunk size is calculated
with:

```
default_chunk_size = max(min(file_size/64, 2M),64K)
```


## Chunk group

Chunks with different sizes are stored in different chunk groups(`ChunkGroup`), e.g.:
- `(0, 4KB]`
- `(4KB, 1MB]`
- `(1MB, 8MB]`
- `(8MB, +oo]`

Opencache has several predefined ChunkGroup and the actual chunk size for an
object chunk size is the smallest predefined chunk size that is no less than the
specified one.


## Object

An object in opencache includes an object header and several chunks.

The header contains application metadata in `JSON` and internal metadata.
The internal metadata contains:
- time when the object is added,
- total size,
- chunk size,
- bitmap of stored chunks
- chunk-id of every stored chunk.


```bob
   .------------.
   |   object   |
   '------------'
      |       |
      |       '-------------.
      | header              |
      |                     |
      v                     |
   .-----------------.      |        .----------.
   | json-meta       |      +------->| chunk-1  |
   | size            |      |        '----------'
   | chunk-bitmap    |      |
   | chunk-ids       |      |        .----------.
   '-----------------'      '------->| chunk-2  |
                                     '----------'
```


Opencache store objects in a persistent map of `<key, header>`.
This map will be loaded in to memory when server starts up.


# Data types

## Chunk group id

`ChunkGroupId` is 2 ascii char in file name, or 1 byte in memory.
`ChunkGroupId` is a encoded representation of the size of chunks in it, similar
to float number representation.

The first char(or 4 higher bits) is the `power`.
The second char(or 4 lower bits) is the `significand`.

```
        power          significand
ascii   0              1
bits    4 higher bits  4 lower bits
```

The chunk size is calculated as follow: `significand * 16^power`

E.g. chunk group id of `48K` chunks is `3c`: `12 * 16^3 = 48K  // c=12`;

Max chunk group id is `ff`: `15 * 16^15 = 15E`

Several ChunkGroupId examples are listed as below:

```
01: 2 * 16^0 = 2        08: 8 * 16^0 = 8
11: 2 * 16^1 = 32       18: 8 * 16^1 = 128
21: 2 * 16^2 = 512      28: 8 * 16^2 = 2K
31: 2 * 16^3 = 8K       38: 8 * 16^3 = 32K
41: 2 * 16^4 = 128K     48: 8 * 16^4 = 512K
51: 2 * 16^5 = 2M       58: 8 * 16^5 = 8M
61: 2 * 16^6 = 32M      68: 8 * 16^6 = 128M
71: 2 * 16^7 = 512M     78: 8 * 16^7 = 2G
```

## Chunk id

`ChunkId` is a `u64`,
in which the most significant byte is `ChunkGroupId`,
the other 7 bytes is mono incremental index in 54-bit int.

```
       <ChunkGroupId> <index>
byte:  [0]            [1, 7]
```

## Key

Key is an arbitrary utf-8 string.


## Access entry

A cache replacement algorithm depends the access log to decide which
object/chunk should be evicted.

Either a `Key` or `ChunkId`, for tracing object access or chunk access
respectively.


# Storage layout

Opencache on-disk storage include:
- Manifest file that track all other kind of on-disk files.
- Key files to store object key and object header.
- Chunk files to store cached file data.


## Manifest file

Manifest is a snapshot of current storage.
I.e., it's a list of files belonging to this snapshot.

```
data_ver: 0.1.0
key_meta_ver: 0.2.0
keys:
  key-wal-1, key-wal-2...
  key-compacted-1, key-compacted-2...
access:
  acc-1, acc-2...
chunk-groups:
  CG-1, CG-2...
chunks:
  chunk-id-1, chunk-id-2...
```

## Object store

Object store is a collection of files to persist the object map.
It includes one active WAL file and several compacted file.

The object header includes:
- The timestamp when this record is added.
- A flag indicate it is an add operation or remove operation.
- Total size of this object.
- User meta data in json.
- ChunkGroup(CG) this object is allocated in.
- `ChunkId` list of the chunks that are stored. In this list ChunkGroup does
    not need to be stored. Because every chunk in an object belong to the
    same ChunkGroup.


## Chunk store

Object content is stored in chunks.
Chunks of the same size belong to the same ChunkGroup.

Every ChunkGroup consists of **one** active(writable) chunk file and several
closed(readonly) chunk files. The active chunk file is append only.

Deleting is implemented with another per-chunk-file WAL file(`ChunkIndex`,
in which chunk index entries are appended one by one:

- For active chunk, ChunkIndex only contains removed chunk index: `(chunk_id, REMOVE)`;
    Because chunk file and ChunkIndex file are not `fsync`-ed when being
    updated.

    When restarting, present chunk ids are loaded from chunk file, while removed
    chunk ids are loaded from ChunkIndex file.

- For closed chunk, ChunkIndex contains present chunk ids and removed chunk
    ids(a chunk is removed after being closed(compact)):
    `(chunk_id, ADD)` or `(chunk_id, REMOVE)`.

    When the server restarts, chunk indexes are loaded just from ChunkIndex
    file.


## Access store

Opencache evicts object or chunk by accessing pattern.
Thus the sequence of recent access has to be persisted.
Otherwise when a cache server restarts, cached data will be evicted
in an unexpected way, due to lacking of accessing information.

Accessing data is tracked in two category: by key and by chunk id.

Accessing data is stored in several append only files, contains key and
chunk-id respectively.

Old access data will be removed periodically.
Access data does not need compact.


```bob
      .----------.
      | Manifest |
      '----------'


  Access   Access    Keys             ChunkGroup-i      ChunkGroup-j   ..
  Key      Chunk

  .----.   .----.    Active           Active
  |key1|   |cid1|    .---------.      .----.  Chunk     ..
  |key2|   |cid2|    | k1,meta |      | c1 |  Index
  | .. |   | .. |    | k2,meta |    .-|-c2 |  .----.
  |CKSM|   |CKSM|    | ..      |    | | .. |<-|cid5|
  |key3|   |cid3|    | CKSM    |    | | .. |  |cid6|
  | .. |   | .. |    | k3,meta |    | '----'  | .. |
  '----'   '----'    | ..      |    |   ..    '----'
    ..       ..      '---------'    |
  .----.   .----.      ..           |
  |keyi|   |cid1|                   |
  |keyj|   |cid2|    Compacted Keys | Compacted Chunks
  | .. |   | .. |    .---------.    |
  |CKSM|   |CKSM|    | k1,meta |    | .----.  Chunk
  |keyk|   |cid3|    | k2,meta |    | | c1 |  Index
  | .. |   | .. |    | ..      |    +-|-c2 |  .----.
  '----'   '----'    | CKSM    |    | | .. |<-|cid5|
                     | k3,meta |    | | .. |  |cid6|
                     | ..  |   |    | '----'  | .. |
                     '---------'    |         '----'
                           |        '---.
                           v            v
                     KeyMeta          Chunk
                     .----------.     .----.
                     | ts       |     |data|
                     | add|rm   |     |CHSM|
                     | size     |     '----'
                     | userdata |
                     | CG       |
                     | chunkids |
                     '----------'
```


# Operations

## Write

A cache server does not need strict data durability, thus when new data is
written(key, chunk, or access data), it does not have to fsync at once.

For different kind of data, the fsync policies are:

- Keys: Keys files are fsync-ed for every `k MB`(where `k` is a configurable parameter). A checksum
  record has to be written before fsync. Since on disk data still has to be
  internally consistent.  The checksum record contains checksum of the `k MB`
  data except itself.

  In other words, Keys file are split into several `k MB` segment.

- Access data is similar to Keys files.

- Chunks is different: Every chunk contains a checksum of itself.
    Chunk files is fsync-ed periodically.


## Update manifest

A write operation does not update `Manifest` file.
`Manifest` is only replaced when files(Key store files, Chunk store files or access store files)
are added or removed(or configuration changed).
E.g., when a new Key file is opened, or a compaction of Chunk files is done.

The Manifest file is named with monotonic incremental integers, e.g.,
`manifest-000021`.

Manifest files can only be added or removed.
When restarting, opencache list all of the manifest file and use the last valid one(by checking the
checksum).


# Memory layout

Keys and Access data are stored in memory for quick access.
Chunks can be optionally cached in memory but it should be OK there is no in-memory chunk-cache.

Keys in memory is just a HashMap: `ObjectMap`.

Key Access data is fed to key-eviction-algorithm.
Chunk Access data is fed to chunk-eviction-algorithm.

When cache server restarts:
All of the keys are loaded into memory.
All Access data are replayed by the two eviction algorithm implementations.



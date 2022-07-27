# API

## Chunk API

ChunkStorage provides chunk access:

```rust
trait ChunkStorage {
    fn add(&mut self, chunk_id, bytes);
    fn remove(&mut self, chunk_id);
    fn read(&mut self, chunk_id) -> Stream<>
}
```

## Object API

ObjectStorage provides object access:

```rust
trait ObjectStorage {
    fn add(&mut self, key: String, size, meta, prefered_chunck_size);
    fn remove(&mut self, key)
    fn access(&mut self, key)
}
```

## CacheServer API

Server level API

```rust
trait Cache {
    /// Write chunks to an object.
    /// Nonexistent object will added implicitly
    fn write(&mut self, key, chunks)

    /// Read specified `range` of bytes in an object
    fn read(&mut self, key, range)
}
```

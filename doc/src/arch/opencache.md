Opencache is a cache server in rust that run in single node mode, and is an object oriented store that provides two API:
range `write` and range `read` and background auto eviction.

Cache cluster management is on client side. Backsourcing is not supported, an application has to explicitly write objects
into it.


# Protocols

- http2 PUT GET
- (maybe) aws-s3
- (maybe) grpc stream

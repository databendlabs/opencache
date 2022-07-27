# Client

Client-side config versioning:
If a new cache node `C` is added to the cache cluster `{A,B}`,
and assuming the cache client uses a consistent hash,
when the new cluster config `{A,B,C}` is published to every client, 1/3 of the data access will encounter a miss.

To upgrade cluster config smoothly, a client needs to access the cache
cluster using two configs: read from the new cluster first, if there is a miss,
read the old cluster, until data migration is done. During this period, there
are two active config `{{A,B},ver=1}` and `{{A,B,C},ver=2}`.

And it is also possible that there are more than two active configs.

Thus for the cache client, it has to:
- Be able to upgrade cluster config on the fly,
- support a **joint** config,
- and be able to retry every config.

A cluster upgrade may looks like this:
- Initially, every cache-client has config `[{{A,B},ver=1}]`;
- A new cache node is added, and the new config is published to every client: `[{{A,B},ver=1}, {{A,B,C},ver=2}]`;
- After a while, config with ver=1 is removed, and another new config is published to every client: `[{{A,B,C},ver=2}]`.

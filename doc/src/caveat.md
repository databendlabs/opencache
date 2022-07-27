# Caveat

## Consistency concern

Cache has consistency issue with the backend storage, i.e., if an object is
changed in the backend store, there is, **NO** way to provide a strong
consistent view on the cache side.
Unless there is a distributed consensus protocol running upon backend store and
cache(which is expensive).

Thus it is recommended to only store object that won't change, i.e, an object on
the backend will only be added and removed, never be modified.

## Space management

It's recommended to set the cache space limit to under 80% of the disk,
since in order to reduce IO, the space of removed chunk won't be reclaim until next `compaction`.

# Storage (getting data in (reads)) and Retrieval (getting data out (writes))

## The Log-Structured Family (LSM - trees)
Build on the concept of Appending-only Log
Instead of overwriting data, its engines sequentially append new data to file.
This concept works well with some kind of magnetic disk, hard disks,... which write sequentially. 
## Hash Indexes
Simplest way to implement a key-value store involves appending key-values pairs to a file (like a CSV)
### Indexing
Everytime we need to read a value, we need to scan the whole file, whtich is inefficient.
To solve this, we can build an in-memory hash table that maps keys to file offsets of the most recent value for that key.
Example:
```plaintext
"Alice" -> 100
"Bob" -> 150
"Alice" -> 200
```
When we get("Alice"), we look up the hash table, find offset 200, and read the value from the file at that offset.
The reason we are not updating the previous value at offset 100 is that we are using an append-only log.
### Compaction
Over time, the file will contain multiple entries for the same key, leading to wasted space.
To address this, we can implement a compaction process that periodically scans the file, removes duplicate entries, and rewrites the file with only the most recent values for each key.
This process helps reclaim space and improve read performance.
We create segments by closing the current log file and starting a new one.
When we compact, we read through all segments, keep only the latest value for each key, and write them to a new segment file.
Example
```Plaintext
Segment 1:
"Alice" -> 100
"Bob" -> 150
Segment 2:
"Alice" -> 200
After compaction:
Segment 3:
"Alice" -> 200
"Bob" -> 150
```
This way, we reduce the number of entries and improve read efficiency.
### Limitations
The hash table must fit in memory, which can be a limitation for large datasets.
Range queries are not efficient since hash tables do not maintain any order among keys. For entries that need to be queried in a range, we would have to scan the entire file.
## String Sorted Indexes (SSTables)

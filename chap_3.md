> Everything you're doing on the internet is all about write and read.
> You play a button then you are write something to the server.
> Then the video is streaming meaning you are reading something from the server.
> As a software engineer, you need to aware of how much read and write your application is doing.
> to determine the best performance strategy for your application.

# Storage is Writes (saving data) and Retrieval is Read(finding data)

There are two main families of storage engines: **log-structured storage engines and page-oriented storage engines**.

## The Log-Structured Family (LSM - trees)
Write-optimized storage engines. For writes, it's hard to beat the performance of simply appending data to a file,
that is the simplest possible way to write data. -> **Immutable data structure**.

There is an important trade-off in storage engines:
    1. Well-chosen indexes speed up read queries.
    2.  But every index adds slows down write.

Build on the concept of Appending-only Log.

**Log**: an append-only sequence of records.

Instead of overwriting data, its engines sequentially append new data to file.

This concept works well with some kind of magnetic disk, hard disks,... which write sequentially. 
## Hash Indexes
Simplest way to implement a key-value store involves appending key-values pairs to a file (like a CSV)

### Indexing
Everytime we need to read a value, we need to scan the whole file, which is inefficient.

To solve this, we can build an in-memory hash table(storage in **RAM**) that maps keys to byte offsets of the most recent value for that key in the data file.
Example:
```plaintext
"Alice" -> 100
"Bob" -> 150
"Alice" -> 200
```
When we get("Alice"), we look up the hash table, find offset 200, and read the value from the file at that offset.

The reason we are not updating the previous value at offset 100 is that we are using an append-only log.
### Compaction
Over time, the file will contain multiple entries for the same key, leading to **wasted memory**.

To address this, we can implement a compaction process that periodically scans the file, removes duplicate entries, 
and rewrites the file with only the most recent values for each key.

This process helps reclaim space and improve read performance.

We create segments by closing the current log file and starting a new one.

When we compact, we read through all segments, keep only the latest update value for each key and throwing away duplicate keys in the log.

You may also merge multiple segments into one brand-new segment during compaction. 
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
Some of the issues in a real implementation:

* File format -> we can use a binary format with fixed-size headers for each record to store key length, value length, and checksum.
* Deleing records -> We can use a special tombstone record to mark deletions.
* Crash recovery -> take a snapshot of the in-memory hash table periodically and write it to disk.
* Partially written records -> Use checksums to detect and ignore corrupted records.
* Concurrency control -> common implementation is to use a single writer thread and multiple reader threads.
This way, we reduce the number of entries and improve read efficiency.
### Advantages
* Append-only and segment merging make writes much faster than random writes and can write data sequentially.
* Segment files are append-only or immutable (a value is not being overwriting), which simplifies crash recovery.
* Fragmentation is not a problem since we merge segments during compaction.
### Limitations
* The hash table must fit in memory, which can be a limitation for large datasets. 
* Range queries are not efficient since hash tables do not maintain any order among keys. For entries that need to be queried in a range, we would have to scan the entire file.
## String Sorted Indexes (SSTables) & Log-Structured Merge Trees (LSM-Trees)
To address the limitations of hash indexes, we can use Sorted String Tables (SSTables), which store key-value pairs in sorted order by key on disk.
* The Algorithm (LSM-Tree):
    1. Writes: Incoming writes are added to an in-memory balanced tree (e.g., Red-Black tree) called a **Memtable**.
    2. Flush: When the Memtable fills up, it is written to disk as a new SSTable segment. Because the tree is already sorted, this write is efficient and sequential.
    3. Reads: The database searches the Memtable first, then the most recent on-disk segment, then older segments.
    4. Compaction: Background processes merge and compact SSTables (similar to the mergesort algorithm).



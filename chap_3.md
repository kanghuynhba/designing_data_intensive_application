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

* File format 
    * &rarr; we can use a binary format with fixed-size headers for each record to store key length, value length, and checksum.
* Deleing records 
    * &rarr; We can use a special tombstone record to mark deletions.
* Crash recovery 
    * &rarr; take a snapshot of the in-memory hash table periodically and write it to disk.
* Partially written records 
    * &rarr; Use checksums to detect and ignore corrupted records.
* Concurrency control 
    * &rarr; common implementation is to use a single writer thread and multiple reader threads.
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
### How's it better?
* Merging segments is simple and efficient since both segments are already sorted. 
    * &rarr; similar to the merge step in the **mergesort algorithm**.
* Range queries are efficient since keys are stored in sorted order. 
    * &rarr; we can use binary search to find the start and end of the range.
    * Example: to find the key `handiwork` and you do know the offset of the key `handbag` and `handsome`, you can start searching because it must appear somewhere between those two keys.
    * &rarr; You're no longer need to store the entire index in memory. Instead, you can load only the relevant segments into memory as needed.
    ### The Algorithm (LSM-Tree):
    1. Writes: Incoming writes are added to an in-memory balanced tree (e.g., Red-Black tree) called a **Memtable**.
    2. Flush: When the Memtable fills up (~MB), it is written to disk as a new SSTable segment. Because the tree is already sorted, this write is efficient and sequential.
        * **WAL (Written Ahead Log)** is often used in conjunction with SSTables to provide durability and crash recovery.
            * The Memtable: You hold a stack of books in your hands (RAM), sorting them alphabetically before putting them on the shelf. If you trip and fall (Crash), you drop the books and lose your sorted order.
            * The WAL: Before you pick up a book, you quickly scribble its name on a piece of scrap paper in your pocket.
            * Recovery: If you trip, you look at the paper to see which books you were holding
            * Discarding: Once the books are safely on the shelf (SSTable), you throw away the scrap paper because you don't need it anymore.
    3. Reads: The database searches the Memtable first, then the most recent on-disk segment, then older segments.
        * Bloom Filters can be used to quickly check if a key might not be in a segment before doing a disk read.
            * Multiple hash functions map keys to bit positions in a bit array.
            * If any of the bits at those positions are 0, the key is definitely not in the set.
                * &rarr; This helps avoid unnecessary disk reads.
    4. Compaction: Background processes merge and compact SSTables (similar to the mergesort algorithm).
        * This reduces the number of segments and removes duplicate keys, keeping only the most recent value for each key.
## B-Tree Family
Page-oriented storage engines. B-trees break the database down into fixed-size blocks or pages, traditionally 4KB in size,
and read or write one page at a time.

This design works more closely to the underlying hardware of hard disks and SSDs.

Each page can be located by an address 
and we can use these references to construct a tree of pages.
### How B-Trees Work
* The basic underlying write operation in a B-tree is to overrite a page on disk with new data.
* The Tree: pages are arranged in a tree. A root page points to child pages based on key ranges.
* Crash Recovery: B-trees often use a Write-Ahead Log (WAL) to ensure durability and recoverability.
### B-Tree Optimizations
* **Copy-on-Write (CoW)**: Instead of overwriting the page and risking corruption (requiring a WAL), some databases write the modified page to a new location and update the parent pointers. This is useful for concurrency (Snapshot Isolation).
* **Abbreviating Keys**: You don't need to store the entire long string in the internal nodes, just enough of the prefix to act as a signpost (boundary). This saves space and increases the Branching Factor (making the tree shorter).
* **Leaf Pointers**: Leaf pages often have references (pointers) to their left and right siblings. This allows for scanning keys in order without jumping back up to the parent.
## Comparing LSM-Trees and B-Trees
|Feature|LSM-Trees (e.g., Cassandra, RocksDB)|B-Trees (e.g., MySQL, PostgreSQL)|
|---|---|---|
|Write Speed|High. Writes are sequential (append-only), which is very fast on magnetic disks and SSDs.|Moderate. Must overwrite pages (random I/O) and write to the WAL.|
|Read Speed|Slower. Must check Memtable, then potentially multiple SSTable segments.|Faster. Keys exist in exactly one place in the tree. Predictable depth.|
|Write Amplification|High. Compaction constantly rewrites data in the background.|Moderate. Writes to WAL + Tree Page + Page Splitting.|
|Fragmentation|Low. Compaction produces tightly packed files.|High. Pages may have unused empty space.|
|Consistency|Weaker. Duplicate keys can exist in different segments before compaction.|Stronger. Keys exist in only one place; easier to implement range locks.|
## Conclusion
Both LSM-trees and B-trees have their strengths and weaknesses. The choice between them depends on the specific requirements of your application, such as read/write patterns, data size, and performance needs.

















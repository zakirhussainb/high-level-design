### Book: Understanding Distributed Systems

#### Chapter 23: Messaging

- **Situation:** Video Encoding

  - The API gateway uploads the video to a file store (e.g., S3).
  - It then writes a message to the channel, including a link to the uploaded
    file.
  - The API gateway responds to the client with a 202 Accepted status, signaling that the request has been accepted for processing but hasn’t completed yet.
  - Eventually, an instance of the encoding service will read the message from the channel and process the video.
  - Crucially, the request (message) is deleted from the channel only when it’s successfully processed. This ensures that if the encoding service fails mid-process, the message will eventually be picked up again and retried by another instance or the same instance after recovery.

- **Question:**
  - It says that the message will be picked up again for processing if the encoding service fails, But how does the message channel know that the request message is processed successfully or not?

## Random Design Questions

### 1: How would you design a distributed backup system that avoids backing up the same data again and again?

### Solution:

Of course. Here’s a breakdown of how to design a **distributed backup system with data deduplication**, structured for a system design interview.

The core idea is to never store the same piece of data twice. We achieve this by breaking files into **content-aware chunks**, identifying them with a unique fingerprint (a hash), and only storing the chunks that the system has never seen before.

#### Core Architecture & Workflow ⚙️

Here's a step-by-step flow of how a file is backed up:

* **Chunking**: The client agent doesn't look at files as a whole. Instead, it uses **Content-Defined Chunking (CDC)**.
    * **What it is**: CDC uses a rolling hash (like the Rabin-Karp algorithm) to scan through the file and find "natural" boundaries in the content to create variable-sized chunks.
    * **Why it's smart**: If you insert a single byte at the beginning of a file, fixed-size chunking would change every single chunk that follows. With CDC, only the chunks directly around the change are affected, maximizing deduplication.

* **Fingerprinting**: For each chunk created, the client agent computes a strong cryptographic hash, like SHA-256. This hash acts as a unique fingerprint for that specific piece of data.
    `Fingerprint = SHA256(ChunkData)`

* **Index Lookup**: Before uploading any data, the client sends the list of all its fingerprints to a central **Metadata Service** (or Indexer).
    * The Metadata Service checks its index to see which of these fingerprints it has already stored from previous backups (from any client).
    * It replies to the client with a list of fingerprints it doesn't have.

* **Data Upload**: The client only uploads the data for the chunks the Metadata Service requested. These unique chunks are sent to a **Chunk Storage layer**.

* **Manifest Creation**: Finally, the client creates a "**manifest**" file for the backup. This is a small file that contains the ordered list of all the fingerprints that reconstruct the original file(s). This manifest is then saved, pointing to the user and the specific backup job.

To restore a file, the system simply retrieves the manifest, fetches each required chunk from the Chunk Store using its fingerprint, and reassembles them in the correct order.

#### Handling Concurrency Trade-Offs ❓

This is a classic distributed systems problem. What happens when two clients generate the same fingerprint for a new chunk and try to upload it simultaneously?

Both clients will query the Metadata Service, which will tell both of them that the chunk is new and needs to be uploaded.

#### Option 1: Pessimistic Locking (Lock Writes) 🔒

In this approach, you prioritize consistency and avoiding redundant work at the cost of speed.

* **How it works**: When the Metadata Service receives a query for a fingerprint it has never seen before, it creates a temporary **lock** on that fingerprint (e.g., using a row lock in a database or a key in a distributed lock manager like ZooKeeper/etcd). It then tells the first client to upload. If a second client asks about the same fingerprint while the lock is active, it's told to wait or that an upload is already in progress. Once the first client confirms the upload is complete, the lock is released, and the index is permanently updated.
* **Pros**:
    * Guarantees a chunk is uploaded only once.
    * Saves network bandwidth and ingest processing on the storage layer.
* **Cons**:
    * **Slower**: Introduces latency as clients may have to wait.
    * **Complex**: Requires a reliable, highly available distributed lock manager, which adds another point of failure.
    * **Reduced Concurrency**: It can become a bottleneck under high load.

#### Option 2: Optimistic, Lock-Free (Clean Up Later) ✨

In this approach, you prioritize speed and simplicity, accepting a small amount of wasted work. This is often preferred in large-scale systems.

* **How it works**: You let the race happen. Both clients are told to upload the new chunk. They both attempt to `PUT` the object to the Chunk Store using the same key (the chunk's fingerprint). Since the data is identical, it doesn't matter which write "wins"—the final result is the same. The object store will simply overwrite the first write with the second identical one.
* **Pros**:
    * **Fast & Highly Concurrent**: No waiting or coordination is required.
    * **Simple & Resilient**: The architecture is simpler without a separate locking service. The system is more resilient as it doesn't depend on a lock manager.
* **Cons**:
    * **Wasted Bandwidth**: The same chunk is uploaded multiple times simultaneously, consuming extra network and ingest resources.

This waste is generally considered an acceptable trade-off, as the simultaneous upload of the exact same new chunk from different clients is a relatively rare event.

For an interview, the **lock-free optimistic approach is usually the better answer**, as it demonstrates an understanding of building scalable, high-throughput systems where avoiding bottlenecks is crucial.

#### Scaling the Index 📈

The Metadata Service, which maps billions or trillions of fingerprints to storage locations, cannot live on a single machine. Here's how to scale it:

* **Bloom Filters**: Before hitting the main index, the client can check against a **Bloom filter**. This is a probabilistic data structure that can quickly tell you if an item is *definitely not* in a set. Clients can download this filter periodically. If the filter says a chunk is new, the client can skip asking the main index and just upload it. This drastically reduces the query load on the index for new data.

---


* **Sharded Key-Value Store**: The primary index must be a distributed database **sharded by the fingerprint hash**. A Distributed Hash Table (DHT) or a sharded NoSQL database (like DynamoDB or Cassandra) is a perfect fit. Consistent hashing is used to map a fingerprint to the specific shard (server) that holds its information, ensuring the load is evenly distributed.

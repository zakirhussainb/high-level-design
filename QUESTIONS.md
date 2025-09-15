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


* **Sharded Key-Value Store**: The primary index must be a distributed database **sharded by the fingerprint hash**. A Distributed Hash Table (DHT) or a sharded NoSQL database (like DynamoDB or Cassandra) is a perfect fit. Consistent hashing is used to map a fingerprint to the specific shard (server) that holds its information, ensuring the load is evenly distributed.

---

### 2: How would you design an idempotent feature-flag system that keeps your UI fast without hammering the backend?

### Approach:

Of course. This is a great question that tests practical system design thinking. Here’s a breakdown of how to build a robust and performant feature-flag system.

The main goal is to deliver the correct flags to the UI instantly without making the user wait or overwhelming the backend with requests. The system must also be idempotent, meaning repeated requests for flags don't change the outcome or cause side effects.

#### Core Architecture & Workflow 🏗️

Here's how the system works from end to end:

* **Backend Flag Service**: This is the source of truth where developers configure flags (e.g., `isNewDashboardEnabled: true`, `searchAlgorithm: 'beta'`) and set targeting rules (e.g., "enable for 50% of users" or "only for users in Canada").

* **Client-Side SDK**: This is a small library integrated into your frontend application (web or mobile). It's responsible for all flag-related logic on the client.

* **Initialization & Fetching**:
    * **On App Load**: The SDK makes a single, asynchronous API call to the backend to fetch all relevant flags for the user.
    * **Graceful Fallback**: If this API call fails (e.g., network error), the SDK falls back to the last known set of flags stored in the cache. If it's the very first time and the cache is empty, it uses a default set of flags bundled with the application code. This prevents the UI from breaking.

* **Smart Caching**:
    * **Storage**: The fetched flags are stored in `localStorage` (for web) to persist them across page reloads and browser sessions.
    * **In-Memory Mirror**: For instant access during the current session, the flags from `localStorage` are loaded into a fast in-memory object (like a JavaScript `Map`). When your application code asks for a flag's value, it's read directly from memory in microseconds.

* **Cache Invalidation**:
    * **Time-To-Live (TTL)**: The flags fetched from the backend come with a TTL (e.g., 5 minutes). The SDK will automatically refetch flags in the background once the TTL expires.
    * **Explicit Events**: The cache is cleared and refetched on key events like user logout (since the next user will have different flags) or when notified of a new deployment.

#### The Big Problem: Handling Stale Cache 😅

This is the critical trade-off question. What if the cache is stale and the UI has an outdated flag, but the user tries to perform a critical action?

**Scenario**: A developer quickly disables a buggy payment feature using a flag `isNewGatewayEnabled`. The backend now has this flag set to `false`. However, a user's browser tab has a cached value of `true`. The user clicks "Pay".

**The Wrong Way**: The UI blindly uses its stale `true` flag and sends the payment request to the new, buggy gateway endpoint. The backend might process it incorrectly or fail.

**The Right Way: Server-Side Verification**
For critical operations, the backend must have the final say. The client should send its current flag state along with the request.

1.  When the SDK fetches flags, it also gets a **version hash** (e.g., `flagsVersion: "v7_abc123"`).
2.  When the user clicks "Pay," the client-side API call includes this version hash in the request headers:
    ```
    POST /api/process-payment
    Headers: { "X-Flags-Version": "v7_abc123" }
    ```
3.  The backend API handler for `/process-payment` first checks this version. It knows the current version is `v8_def456`.
4.  Since the versions don't match, the backend rejects the request with a specific error, like `409 Conflict` and a message `{"error": "flags_outdated"}`.
5.  The client-side SDK catches this specific error, forces an immediate refetch of the flags, and can then prompt the user to retry the action.

This pattern ensures that critical actions are always executed against the server's source of truth, gracefully handling any cache staleness.

#### What Else Could Go Wrong? 🤔

Even with a solid design, you can run into practical issues:

* **"Flash of Incorrect Content" (FOIC)**: On the initial load, the UI might render with default flags for a split second before the fresh flags arrive from the API, causing a visual flicker.
    * **Solution**: Display a loading state or skeleton UI for the components affected by flags until the initial fetch is complete. This is the most common and effective solution.

* **Changing User Context**: What if flags depend on user attributes that change mid-session (e.g., a user upgrades their subscription)?
    * **Solution**: The application needs to notify the feature-flag SDK of this context change. The SDK would then trigger a refetch with the new user attributes to get the correct set of flags.
        `featureFlagSDK.updateContext({plan: 'premium'})`

* **Performance on Load**: The initial flag fetch is in the critical path of your application startup.
    * **Solution**: Use a **Content Delivery Network (CDN)** to serve the flag delivery API. This caches flag configurations at edge locations globally, ensuring low latency for all users.

# System Design Interview Questions: Software Testing

Here are some interview questions and answers based on the provided document, focusing on testing strategies for large-scale systems.

---

## Question 1: Testing a Critical Microservice

**Scenario:** You are tasked with designing the testing strategy for a new, critical microservice in a large e-commerce platform. This service is responsible for processing orders. It interacts with a central database, an internal inventory service, and a third-party billing API.

**Question:** How would you approach testing this service? Describe the different types of tests you would implement, their proportions, and the rationale behind your strategy.

### Solution

My testing strategy would be based on the **Test Pyramid** model to balance confidence, speed, and cost. The goal is to have a large number of fast, reliable tests at the base and a small number of slower, broader tests at the top.



* **Unit Tests (Largest Quantity):** I would start with a comprehensive suite of unit tests.
    * **Scope:** These tests would validate small, individual parts of the codebase, like a single class or function, in isolation. For example, testing the logic for order validation or price calculation.
    * **Implementation:** They would focus on testing the public interfaces of the code, verifying behavior and state changes. These tests would be **small**, running in a single process without any blocking calls or I/O, making them fast and deterministic.

* **Integration Tests (Medium Quantity):** Next, I would implement integration tests to verify the service's interactions with its external dependencies.
    * **Scope:** I'd use "narrow integration tests". For instance, one set of tests would verify that the service's data store adapter can correctly communicate with the database. Another would test the interaction with the inventory service, and a third for the billing API.
    * **Implementation:** To keep these tests fast and reliable, I would use **test doubles** where appropriate.
        * For the database, I'd use an in-memory fake if available, as it closely mimics the real implementation without the overhead of network calls.
        * For the internal inventory service and the third-party billing API, I would use mocks combined with **contract tests**. The contract defines the expected request/response pairs, ensuring our mocks are valid and that our service can communicate correctly with the live APIs without actually calling them during the test. This avoids slow, flaky, and potentially costly test runs (e.g., creating real transactions).

* **End-to-End Tests (Smallest Quantity):** Finally, I would have a very small number of end-to-end (E2E) tests.
    * **Scope:** These tests are necessary to validate user-facing scenarios that span multiple services.
    * **Implementation:** I would frame these as **user journey tests**. For example, a single E2E test might simulate a user creating an order, modifying it, and then canceling it. This approach is more efficient than writing separate E2E tests for each step. Because E2E tests are slow, brittle, and hard to debug, their number must be minimized.

This layered strategy ensures we catch most bugs early with fast, cheap unit tests, while still having confidence that the integrated components and the entire user journey work as expected.

---

## Question 2: The Problem with Test Doubles

**Scenario:** During a design discussion, a colleague suggests that to make your team's integration tests faster, you should replace all real dependencies (databases, other services) with mocks.

**Question:** What are the potential dangers of relying too heavily on mocks and stubs? How can you mitigate these risks while still keeping tests efficient?

### Solution

While mocks and stubs can make tests faster and less flaky by reducing a test's size and eliminating external dependencies, over-reliance on them presents a significant risk: **they don't behave exactly like the real implementation**. The less a test double resembles the real dependency, the less confidence we can have in the test's usefulness. This can lead to a false sense of security where tests pass, but the application fails in production.

Here's a breakdown of the issues and mitigation strategies:

* **The Problem with Mocks/Stubs:**
    * **Weak Resemblance:** Mocks and stubs are considered "last-resort" options because they offer the least resemblance to the actual implementation. They might not correctly model the dependency's latency, error modes, or subtle behaviors.
    * **Brittle Tests:** Tests using mocks often verify the *interactions* between objects rather than the resulting state. This makes the tests brittle; they can break even when a refactoring doesn't change the actual behavior, violating a core principle of good testing.

* **Mitigation Strategies:**
    1.  **Prefer Real Implementations When Possible:** The first rule is to avoid a double if the real implementation is fast, deterministic, and has few dependencies of its own.
    2.  **Use High-Fidelity Fakes:** If a real implementation isn't an option, the next best choice is a **fake**. A fake is a lightweight but working implementation of the dependency, such as an in-memory database. Ideally, this fake is maintained by the same team that owns the dependency, ensuring it stays up-to-date.
    3.  **Combine Mocking with Contract Tests:** When mocks are unavoidable for external services, the best way to mitigate the risk is with **contract tests**.
        * A contract test defines a strict request and its expected response for an external dependency (e.g., an HTTP request/response pair for a REST API).
        * Our service's integration test then uses this contract to mock the dependency.
        * Crucially, the external dependency's own test suite uses the *same contract* to run a test against their live service, guaranteeing that the response we are mocking is accurate and up-to-date. This ensures our mock doesn't drift from reality.

By following this hierarchy—**Real > Fake > Mock with Contract**—we can write tests that are both efficient and provide strong guarantees that our service interacts correctly with its dependencies.

---

## Question 3: High-Stakes System Verification

**Scenario:** You are designing a critical, zero-downtime data migration process to move a service from one key-value store (X) to another (Y). The plan involves dual-writing to both stores for a period, backfilling old data, and then switching reads to the new store.

**Question:** This process is prone to subtle bugs, like race conditions, that are very difficult to catch with traditional testing. How could you verify the correctness of this migration *architecture* before implementation to prevent data inconsistency?

### Solution

For a high-stakes process like this, where subtle concurrency bugs can lead to data corruption, traditional testing is insufficient because it can only cover failures developers can imagine. A better approach is **Formal Verification**, which allows us to detect architectural flaws before writing any implementation code.

Here’s how I would apply this concept:

1.  **Create a Formal Specification:** I would write a high-level mathematical description of the migration process using a formal language like **TLA+**. Companies like Amazon and Microsoft use TLA+ for precisely this kind of complex distributed system design. The specification describes all possible states and behaviors of the system. This act of formally specifying the system itself helps clarify our reasoning and can uncover issues.

2.  **Define System Properties:** The main goal is to verify that the specification satisfies key system properties, specifically **safety** and **liveness**.
    * **Safety Property (Invariant):** An essential safety property here would be that "for any given key, the value in store X must always be consistent with the value in store Y after the backfill is complete." A safety property asserts that something bad never happens.
    * **Liveness Property:** A liveness property would be that "the migration process eventually completes successfully".

3.  **Use Automated Model Checking:** With the specification and properties defined, I would use a **model checker** for TLA+. This tool algorithmically explores all possible states and behaviors the system can enter, something humans are terrible at, especially with concurrency.

4.  **Identify and Fix Flaws:** The model checker would likely identify critical flaws in the initial design. For example:
    * **Race Condition:** As shown in the provided document, if two service instances write different values for the same key simultaneously, network latency could cause the writes to arrive in different orders at stores X and Y. This would leave the two data stores in permanently inconsistent states, violating our safety property.
    * **Error Trace:** The beauty of model checking is that when it finds a flaw, it provides a specific **error trace**—a sequence of steps showing exactly how the system gets into the bad state. This makes the abstract bug concrete and easy to understand.



By using formal verification, we can test our architectural decisions and find deep design flaws when they are still cheap to fix, gaining a much higher degree of confidence than is possible with testing alone.

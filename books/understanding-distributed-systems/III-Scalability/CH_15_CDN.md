# 📘 System Design Notes: Chapter 15 – Content Delivery Networks (CDNs)

---

## 🔑 What is a CDN?

- A **Content Delivery Network (CDN)** is a **decentralized network of caching servers** (also known as reverse proxies).
- Designed to overcome the limitations of the public internet by placing **content closer to users**.
- When a client requests a URL:
  - The CDN server intercepts it.
  - If cached → content is returned immediately.
  - If not → CDN fetches it from the **origin server**, caches it, and returns it.
- **Examples**:
  - 🟡 **Amazon CloudFront**
  - 🔵 **Akamai**

---

## 🌐 15.1 Overlay Network

### 🧩 CDN as a Network

- Not just about **caching**; the **network layer** of a CDN is equally critical.
- Public internet uses **BGP (Border Gateway Protocol)** which:
  - Prioritizes **fewer hops**, not **lower latency** or **bandwidth efficiency**.
  - Often leads to **suboptimal routing paths**.

### 🌍 What is an Overlay Network?

- A **virtual network** built on top of the public internet.
- Allows:
  - **Custom routing** optimized for latency & congestion.
  - Use of **private backbone networks** for more efficient data flow.
- Optimizations include:
  - Monitoring **real-time health** of links.
  - Avoiding congested or failing paths.

### 🗺️ Geographical Distribution

- CDN edge servers are **strategically placed** in various regions.
- Closer proximity to clients → reduces **network latency** (often below 50ms).
- **Physical constraints** like the speed of light impose latency floors over long distances.

### 🧭 Client Routing

- Uses **Global DNS Load Balancing**:
  - DNS resolves domain to IPs of nearby, healthy CDN edge servers.
  - Considers:
    - Client location (via IP geolocation).
    - Server health and response time.
    - Current network congestion.

### 🧵 Connection Management

- CDNs keep **persistent TCP connections** between nodes.
- Benefits:
  - Avoids **TCP handshake overhead** on repeated requests.
  - Uses **optimized TCP window sizes** for bandwidth utilization.
  - Increases throughput and reduces packet loss.
- Especially valuable for **real-time content delivery**.

### 🔒 Frontline Defense & Dynamic Content

- CDNs can handle **uncacheable dynamic content**.
- In this role, the CDN is a **front proxy** to your application.
- Also serves as a **DDoS protection layer**:
  - Filters malicious requests.
  - Offloads attack traffic from origin.

---

## 🧠 15.2 Caching in CDNs

### 🏗️ Multi-Layer Architecture

#### Layer 1: Edge Clusters

- Located close to clients.
- Store **frequently accessed** content.
- Ideal for static files (images, videos, scripts, etc.).

#### Layer 2: Intermediate Clusters

- Fewer in number.
- Store **less frequently accessed** content.
- Reduce load on the origin and improve performance for regional users.

#### Layer 3: Origin Servers

- Main application or database server.
- Accessed only on **cache misses** from upper layers.

---

### ⚖️ Cache Hit Ratio (CHR)

- **CHR** = Percentage of client requests served from cache.
- Higher CHR → Better performance & lower origin load.
- **Tradeoff**:
  - More edge clusters → Better global coverage, but lower CHR.
  - Fewer clusters → Higher CHR, but more latency for remote users.

**Solution**: Use **multi-layer caching hierarchy**.

---

### 📂 Content Partitioning (Sharding)

- No single CDN server can cache all data.
- Data is **sharded** (partitioned) across multiple machines.
  - Each server holds a **subset** of content.
- Increases scalability and parallelism.

---

## 🔁 End-to-End Request Flow (Simplified)

Client → DNS Resolution → Nearest Edge Server →
[Cache Miss] → Intermediate Cache →
[Cache Miss] → Origin Server

markdown
Copy
Edit

---

## 📌 Glossary of Key Terms

| Term                          | Definition                                                                            |
| ----------------------------- | ------------------------------------------------------------------------------------- |
| **CDN**                       | A distributed system of proxy servers that deliver content based on client proximity. |
| **Overlay Network**           | A logical network built on top of the internet for optimized routing.                 |
| **Edge Server**               | A CDN server close to end-users that handles requests.                                |
| **Persistent TCP Connection** | A connection that remains open for multiple requests.                                 |
| **Cache Hit**                 | Data found in the cache; no need to access the origin server.                         |
| **Cache Miss**                | Data not in cache; fetched from origin server.                                        |
| **Cache Hit Ratio (CHR)**     | Proportion of requests served from cache vs total requests.                           |
| **Global DNS Load Balancing** | DNS-based mechanism to route clients to optimal servers.                              |
| **DDoS Protection**           | Techniques used to prevent distributed denial-of-service attacks.                     |

---

## 🧩 Summary of Key Concepts

- CDNs improve **speed**, **availability**, and **scalability**.
- They optimize both **data delivery** and **network routing**.
- Provide **caching** + **protection** (e.g., DDoS mitigation).
- Employ a **multi-layered** and **partitioned** infrastructure.
- Enable efficient delivery of both **static** and **dynamic** content.

# Load Balancer - High Level Design (HLD) Study Guide

## Introduction
Modern applications like Amazon, Netflix, Uber, and Instagram serve millions of users every day. If every request reaches a single server, that server eventually becomes overloaded, leading to:

- Slow response times
- High CPU usage
- Memory exhaustion
- Request failures
- Complete downtime

To solve this problem, applications are deployed on multiple servers, and a **Load Balancer (LB)** is introduced to distribute incoming traffic efficiently.

A Load Balancer is one of the first components added while designing scalable and highly available systems.

---

## What is a Load Balancer?
A Load Balancer is a server (or software) that sits between clients and backend servers.Instead of users directly contacting application servers, all requests first reach the Load Balancer. The Load Balancer decides which backend server should process the request.

```text
              Users
                 |
      ----------------------
      |   Load Balancer    |
      ----------------------
        /       |       \
       /        |        \
   App-1     App-2     App-3
```

The client only knows one public endpoint. Example: ```api.amazon.com```. The Load Balancer hides backend servers from users.

---

## Why Do We Need a Load Balancer?
Imagine this architecture:
```
Users
  |
Server
```
Everything works well until traffic increases.

Suppose:

- 1 million users
- 5000 requests/sec

**One server cannot handle all requests. Problems:**
- CPU overload
- High latency
- Memory exhaustion
- Crashes
- Downtime

Adding multiple servers solves capacity issues, but introduces another problem:
```
Server A

Server B

Server C
```

#### How does a client know which server to use?
Without a Load Balancer:

```
Everyone chooses Server A
↓
Server A → Overloaded
Server B → Idle
Server C → Idle
```

A Load Balancer distributes requests evenly.

---

## Basic Architecture

```text
          Internet
              |
             DNS
              |
        Load Balancer
      /     |       \
    /       |        \
App-1    App-2     App-3
      \     |       /
          Database
```

Every request follows the same path.

The backend servers remain hidden.

---

## Problems Solved by Load Balancers

### 1. Scalability
Instead of:
```
Users
↓
1 Server
```

**we use:**

```
Users
↓
Load Balancer
↓
10 Servers
```

Traffic gets distributed across multiple servers.

---

### 2. High Availability

Suppose:
```
App-1
App-2
App-3
```

Now App-2 crashes.
- Without LB: Requests fail.
- With LB:
```
App-2 removed
↓
Traffic redirected
↓
App-1
App-3
```
Users continue using the application.

---

### 3. Better Resource Utilization

Without LB:
```
App-1 → 90%
App-2 → 10%
App-3 → 5%
```

With LB:
```
App-1 → 35%
App-2 → 33%
App-3 → 32%
```

Resources are utilized efficiently.

---

### 4. Horizontal Scaling
Need more capacity? Simply add:
```
App-4
App-5
```
No client changes required.

---

## Request Flow
```
User
↓
DNS
↓
Load Balancer
↓
Application Server
↓
Database
↓
Application Server
↓
Load Balancer
↓
User
```

The client never communicates directly with backend servers.

---

## Load Balancing Algorithms

---

### 1. Round Robin
Requests are distributed sequentially.
Servers:
```
S1
S2
S3
```

Requests:
```
R1 → S1
R2 → S2
R3 → S3
R4 → S1
R5 → S2
R6 → S3
```

#### Advantages
- Very simple
- Equal distribution

#### Disadvantages
Assumes every request requires equal processing time.

---

### 2 Weighted Round Robin
Powerful servers receive more requests.

**Example:**

| Server | Weight |
|---------|--------|
| S1 | 4 |
| S2 | 2 |
| S3 | 1 |

Traffic pattern:
```
S1
S1
S1
S1
S2
S2
S3
```

Useful when servers have different hardware capacities.

---

### 3. Least Connections
Load Balancer checks active connections.
**Example:**
```
S1 → 150 users
S2 → 20 users
S3 → 80 users
```
Next request goes to:
```
S2
```

Useful for:

- WebSockets
- Chat
- Video Streaming

---

### 4. Least Response Time
Choose the fastest server.
**Example:**

```
S1 → 15 ms
S2 → 25 ms
S3 → 300 ms
```

Next request:
```
S1
```
Ideal for latency-sensitive systems.

---

### 5. IP Hash

Hash: ```Client IP```

Same IP always reaches the same server.

##### Useful for:
- Shopping carts
- Sticky Sessions

---

### 6. Consistent Hashing
Instead of: ```hash(UserID) % N```

Servers are placed on a logical ring.

```
         S1
     /         \
 UserA       UserB
    |           |
 S3            S2
```

##### Benefits:

- Minimal remapping
- Cache preservation
- Better scalability

##### Used in:

- Redis Cluster
- Cassandra
- DynamoDB

---

## Health Checks
The Load Balancer continuously verifies server health.
**Example:**
``` GET /health```

Healthy: ```200 OK```

Unhealthy: 
```
500
Timeout
Connection Refused
```

Unhealthy servers are removed automatically.

---

## Active Health Checks

```
LB
↓
Ping Server
↓
Response?
```

Runs periodically.

---

## Passive Health Checks
LB monitors production traffic.If failures exceed a threshold, the server is removed.
 
---

## Sticky Sessions
Suppose user logs in. Session stored on: ```App-1 Memory```

Next request goes to: ```App-2```

Problem: ```User appears logged out```

**Solution:**
Sticky Session
```
Same User
↓
Same Server
```

**Implemented using:**

- Cookies
- IP Address

**Drawback:** If the server crashes, the session is lost.
---

## External Session Storage
Instead of storing sessions in application memory:
```
App-1
App-2
App-3
↓
Redis
```

#### Advantages:

- No Sticky Sessions
- Better scaling
- Better fault tolerance

---

## Layer 4 vs Layer 7 Load Balancer

| Layer 4 | Layer 7 |
|-----------|-----------|
| TCP/UDP | HTTP/HTTPS |
| Faster | Smarter |
| Routes using IP + Port | Routes using URL/Header |
| Cannot inspect HTTP | Can inspect HTTP |

**Example:** 
- Layer 4:
```
TCP
↓
Server
```

- Layer 7:
```
GET /payments
↓
Payment Service
GET /chat
↓
Chat Service
```

---

## Reverse Proxy vs Load Balancer
#### Reverse Proxy
- SSL Termination
- Compression
- Caching
- Security
- Request Routing

#### Load Balancer
- Distributes traffic
- Removes unhealthy servers
- Balances requests

**Many tools perform both roles:**
- NGINX
- HAProxy

---

## Avoiding Single Point of Failure

Bad Architecture
```
Users
↓
Load Balancer
↓
Servers
```

If LB crashes:
Entire application becomes unavailable.
Better Architecture
```
             DNS
              |
      ----------------
      |              |
    LB-1          LB-2
      |              |
      ----------------
             |
       Application Servers
```

Two Load Balancers can operate in:

- Active-Active
- Active-Passive

---

## Real World Examples

#### Netflix
```
Users
↓
DNS
↓
Global Load Balancer
↓
Regional Load Balancer
↓
Microservices
```

Traffic is routed to the nearest healthy region.

---

### Amazon

```
Users
↓
Application Load Balancer
↓
EC2 Instances
↓
RDS / DynamoDB
```

Servers can be added or removed without changing the public endpoint.

---

## Common Interview Questions

### Q1. What is a Load Balancer?
A Load Balancer distributes incoming requests across multiple backend servers to improve scalability, availability, and reliability.

---

### Q2. Why do we need a Load Balancer?
To prevent a single server from becoming overloaded and to allow applications to scale horizontally.

---

### Q3. What happens when one server crashes?
The Load Balancer detects the failure using health checks, removes the server from the pool, and routes traffic only to healthy servers.

---

### Q4. Why is Round Robin not always ideal?
Because it assumes every request takes the same amount of time. Long-running requests can overload one server.

---

### Q5. When should Least Connections be used?
When request durations vary significantly, such as WebSockets, chat systems, or streaming services.

---

### Q6. What are Sticky Sessions?
A mechanism that routes requests from the same client to the same server to preserve session state.

---

### Q7. Why are Sticky Sessions discouraged?
Because they reduce scalability and create issues when a server fails. External session storage like Redis is preferred.

---

### Q8. Difference between Layer 4 and Layer 7 Load Balancers?
- **Layer 4** works on TCP/UDP.
- **Layer 7** understands HTTP and can route requests based on URLs, headers, cookies, and hostnames.

---

### Q9. Difference between Reverse Proxy and Load Balancer?
- **Reverse Proxy** focuses on request processing (SSL termination, caching, compression).
- **Load Balancer** focuses on distributing traffic across backend servers.
Many modern tools combine both functionalities.

---

### Q10. Why can't DNS replace a Load Balancer?
DNS can return multiple IP addresses but cannot monitor server health or make per-request routing decisions.

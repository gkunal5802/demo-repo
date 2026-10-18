# High Availability
High Availability is the ability of a system to continue serving user requests with minimal downtime even if hardware, software, or network failures occur. It is achieved by eliminating single points of failure through redundancy, replication, automatic failover, and health monitoring.

## Availability Percentage
Availability is measured in percentages.

#### Example:

```Availability = Total Uptime/Total Time```

```text
System runs: 365 days
Down for: 1 day
Availability = 364 / 365 ≈ 99.73%
```

Companies often express this as "number of nines."

**The Famous Nines**
| Availability | Downtime per Year |
| ------------ | ----------------- |
| 99%          | ~3.65 days        |
| 99.9%        | ~8.76 hours       |
| 99.99%       | ~52.6 minutes     |
| 99.999%      | ~5.26 minutes     |

This is a favorite interview question because it illustrates how each additional "9" dramatically reduces the allowed downtime and increases engineering complexity.

## Why 100% Availability is Impossible OR Why can't companies provide 100% uptime?
Because:
Hardware eventually fails.
Software contains bugs.
Operating systems need updates.
Networks experience outages.
Cloud providers can have regional failures.
Databases require maintenance.
Humans make deployment mistakes.

Designers therefore target a practical Service Level Objective (SLO), such as 99.9% or 99.99%, instead of absolute perfection.

## Why More "9s" Cost More
Going from 99% to 99.9% isn't just a small improvement—it requires significantly more infrastructure and operational sophistication.

For example:
99%: A single server with backups may be acceptable.
99.9%: Multiple application servers behind a load balancer.
99.99%: Database replication, automatic failover, monitoring, and deployments across multiple availability zones.
99.999%: Often requires multi-region deployments, advanced disaster recovery, and extensive automation.

Each additional "9" reduces acceptable downtime but increases infrastructure cost and system complexity.

## What causes downtime?
Almost anything can fail.

- Application Server
- Database
- Load Balancer
- Network Switch
- Power Supply
- Disk
- Entire Data Center
- DNS
- Cache

## Pillars of High Availability
Instead of thinking only about Active-Active and Active-Passive, think of HA as several building blocks:
```css
                High Availability
                       │
      ┌────────────────┼─────────────────┐
      │                │                 │
 Redundancy       Health Checks      Automatic Failover
      │                │                 │
 Replication     Monitoring         Load Balancing
      │                │                 │
 Multiple AZ     Retry Logic        Disaster Recovery
```

### Two Ways to Achieve HA

There are two deployment models:
1. Active-Passive
2. Active-Active

## Active-Passive Architecture
Only one site serves production traffic.

```css
      Users
        │
        ▼
  Active DC (Primary)
        │
  Primary Database
        │
Replication
        │
  Passive DC
        │
  Replica Database
```

The passive site is continuously updated but does not normally serve traffic. If the active site fails, Passive DC becomes Primary.

### Components
- Primary Application
- Primary Database
- Replica Database
- Replication
- Health Check
- Failover Mechanism

## Database Replication
Traditional relational databases (such as PostgreSQL, MySQL, Oracle, and SQL Server) are commonly deployed in a single-primary (leader-follower) configuration. The primary accepts writes, while replicas receive changes through replication and are mainly used for reads or failover.

"In a typical Active-Passive architecture, only the primary database accepts writes. Replicas are synchronized from the primary and are often used for read traffic and failover."

### Failover

When the primary dies
```css
Primary
↓
Health Check
↓
Failure detected
↓
Promote Replica
↓
Traffic switched
```
This process is called Failover. "Active-Passive introduces failover time, which may range from seconds to minutes depending on the infrastructure and automation."

### Advantages of Active-Passive
- Simple architecture.
- Easier consistency.
- Easier deployment.
- Lower operational complexity.
- Predictable database writes.

### Disadvantages

1. Underutilized resources:
The passive site is mostly idle.

CPU
Primary : 80%
Passive : 5%
You're paying for infrastructure that isn't serving production traffic.

2. Failover delay:
Traffic pauses during promotion and redirection.

3. Cross-region latency (if applicable)"
This is more precise than saying "latency because passive writes to active." If application instances in another region must communicate with the primary database across regions, network latency increases.

## Active-Active Architecture
Every data center serves users.
```css
      Global Load Balancer
    /                 \
  DC1                 DC2

Read/Write          Read/Write
```

Every site is active.

#### Database
Usually NoSQL or Distributed SQL. Supports writes from multiple regions more naturally than traditional single-primary deployments.

Examples include systems designed for multi-region replication and conflict handling.

Replication: ```DC1 ⇄ DC2 ⇄ DC3```
Everyone keeps synchronizing.

Biggest Challenge: Conflicts.

Suppose: 
User updates profile

```yaml
Delhi
Name = Rahul
```
At the same time

```yaml
Mumbai
Name = Rohit
```
Which one wins?

This is called a write conflict.

### Conflict Resolution Strategies
* Last Write Wins (LWW): Newest timestamp wins. Simple but can overwrite earlier updates.
* Version Numbers: Every update increments a version. Conflicts can be detected and resolved based on version history.
* Vector Clocks: Used by systems like Dynamo-inspired databases to detect concurrent updates.

### Advantages
- No downtime if one data center fails.
- Better global latency because users connect to the nearest region.
- Better resource utilization.
- Higher scalability.

### Disadvantages
- Conflict resolution.
- Eventual consistency (in many designs).
- More operational complexity.
- Harder debugging.
- More expensive infrastructure.

## Active-Passive vs Active-Active

| Feature              | Active-Passive                          | Active-Active                                                  |
| -------------------- | --------------------------------------- |-------------------------------------------------------------- |
| Active sites         | One                                     | All                                                            |
| Write location       | Primary only (typical)                  | Multiple sites                                                 |
| Read location        | Primary and often replicas              | Any site                                                       |
| Failover             | Required                                | Usually seamless                                               |
| Resource utilization | Lower                                   | Higher                                                         |
| Complexity           | Low                                     | High                                                           |
| Consistency          | Easier (stronger by default)            | More challenging                                               |
| Conflict handling    | Rare                                    | Essential                                                      |
| Typical use cases    | Banking, traditional enterprise systems | Global-scale services like social media, messaging, e-commerce |

### Interview Framework

Whenever you're asked to design a 99.99% available or resilient system, follow this sequence:
```text
1. Identify every Single Point of Failure.
    ↓
2. Add redundancy.
    ↓
3. Add health checks.
    ↓
4. Add automatic failover.
    ↓
5. Replicate data.
    ↓
6. Deploy across multiple Availability Zones.
    ↓    
7. Consider multi-region deployment.
    ↓
8. Discuss Active-Passive vs Active-Active.
    ↓
9. Explain monitoring and disaster recovery.
```

## Interview Questions:

* What is High Availability, and how is it different from Resilience?
* What is a Single Point of Failure? Give examples.
* Why can't a single server provide high availability?
* Explain Active-Passive architecture with its advantages and disadvantages.
* Explain Active-Active architecture and when you would choose it.
* How is failover implemented in practice?
* What role do health checks play in High Availability?
* Why are replicas commonly used for reads in relational databases?
* How would you resolve write conflicts in an Active-Active system?
* Why is Active-Active more difficult than Active-Passive?
* What happens if an entire data center goes down?
* What is the difference between High Availability and Disaster Recovery?
* How do Availability Zones contribute to High Availability?
* What are MTTR and MTBF, and why do they matter?
* How would you design a payment system with 99.99% availability?

1. What are MTTR and MTBF?
- MTBF (Mean Time Between Failures) measures how long a system operates before failing. Higher MTBF indicates better reliability.
- MTTR (Mean Time To Repair or Recover) measures how quickly a failed system is restored. Lower MTTR improves availability.

Availability can be approximated as:
Availability ≈ MTBF / (MTBF + MTTR)

Therefore:
- Increase MTBF by improving reliability.
- Decrease MTTR through automation, monitoring, and failover.

#### Which metric is more important for High Availability?
Both matter, but reducing MTTR often has a more immediate impact because faster recovery directly reduces downtime.

2. How would you design a payment system with 99.99% availability?
- For a payment system, I would focus on eliminating every critical Single Point of Failure while preserving data correctness.

**My approach would include:**
- Multiple application servers behind a load balancer so requests continue even if one server fails.
- Database replication with automatic failover, using a single primary for writes to maintain consistency.
- Deploy across multiple Availability Zones to survive data center failures.
- Use health checks and automatic failover for servers and databases.
- Use idempotency keys so retries do not create duplicate payments.
- Introduce message queues for asynchronous processing where appropriate.
- Add monitoring, alerting, logging, and distributed tracing.
- Implement backups and disaster recovery procedures for catastrophic failures.

For payments, I would generally prefer Active-Passive for the database layer because consistency is more important than accepting writes from multiple regions simultaneously. If global deployments are required, I'd carefully design regional routing and conflict avoidance rather than relying on unrestricted multi-region writes.
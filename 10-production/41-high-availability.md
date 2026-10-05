# Lesson 41 — PostgreSQL High Availability

## Goal of This Lesson

High Availability (HA) is the architecture and operational process used to keep PostgreSQL available when infrastructure or database components fail.

Replication from Lesson 40 is an important building block, but:

> **Replication alone is not High Availability.**

A complete HA system must solve:

~~~text
Replication
Failure detection
Failover decision
Standby promotion
Client/application rerouting
Old-primary fencing
Recovery and rejoining
Monitoring
~~~

---

# 1. What Is High Availability?

High Availability means designing a system to minimize service interruption when failures occur.

Basic architecture:

~~~text
Application
    ↓
Primary PostgreSQL
    ↓ WAL
Standby PostgreSQL
~~~

If the primary fails, the system should be able to move service to a healthy standby safely.

---

# 2. Single Point of Failure

Without redundancy:

~~~text
Application
    ↓
PostgreSQL
    ✗
Database unavailable
~~~

The single database server is a **single point of failure**.

HA attempts to remove or reduce critical single points of failure.

---

# 3. Replication Is Only the First Step

Suppose you have:

~~~text
Primary → Standby
~~~

and the primary crashes.

Questions remain:

~~~text
Who detects the failure?
Who decides the primary is really unavailable?
Who promotes the standby?
How does the app discover the new primary?
How do we prevent the old primary from accepting writes?
What happens when the old primary returns?
~~~

Those questions are the heart of HA.

---

# 4. HA Failure Flow

Core mental model:

~~~text
Primary failure
      ↓
Failure detection
      ↓
Failover decision
      ↓
Select healthy standby
      ↓
Promote standby
      ↓
Fence old primary
      ↓
Redirect application
      ↓
Restore redundancy
~~~

Every production HA design should have clear answers for these stages.

---

# 5. Failure Detection

An HA system must determine whether the primary is healthy.

Signals can include:

~~~text
database connection checks
SQL health checks
process health
network reachability
replication status
host/VM health
storage health
~~~

A single failed ping is generally not enough evidence to safely declare a database dead.

---

# 6. Why Failure Detection Is Hard

Consider:

~~~text
Primary is healthy
but
network between HA controller and primary is broken
~~~

The controller may think:

~~~text
Primary is dead
~~~

while the primary may still be accepting writes from applications.

If a standby is promoted without protecting against the old primary, you can create split brain.

---

# 7. Split Brain

Split brain means multiple nodes believe they are the writable primary.

~~~text
          ┌→ Primary A → writes
Clients ──┤
          └→ Primary B → different writes
~~~

Now the database histories can diverge.

This is one of the most dangerous HA failure modes.

---

# 8. Fencing

Fencing means ensuring the old/failed primary cannot continue accepting conflicting writes before or while another node becomes authoritative.

Conceptually:

~~~text
Old Primary
    ↓
disable / isolate / revoke write path
    ↓
Promote Standby
~~~

Fencing can involve infrastructure, orchestration, network, storage, or process controls depending on the HA system.

Key interview point:

> **Promotion without reliable fencing can create split brain.**

---

# 9. Failover

Failover is switching database service from a failed/unhealthy primary to a standby.

~~~text
Before
Application → Primary A
                ↓
             Standby B

Failure
Primary A ✗

After failover
Application → Primary B
~~~

Failover may be manual or automated.

---

# 10. Switchover vs Failover

These are related but different.

~~~text
Failover
→ usually unplanned response to failure

Switchover
→ planned role change between healthy nodes
~~~

A switchover may be used for:

~~~text
maintenance
server replacement
planned upgrade work
testing HA procedures
~~~

---

# 11. Promotion

A PostgreSQL standby normally operates in recovery/read-only standby mode.

Promotion changes it into a writable primary.

High-level:

~~~text
Standby
   ↓ promotion
New Primary
   ↓
accept writes
~~~

PostgreSQL provides mechanisms such as `pg_ctl promote` and `pg_promote()` for promotion, but HA orchestration normally coordinates much more than the promotion command itself.

---

# 12. Application Rerouting

After promotion, the application must connect to the new primary.

Bad architecture:

~~~text
Application
    ↓
hard-coded old-primary IP
~~~

Better HA architecture uses an indirection layer such as:

~~~text
stable DNS name
database proxy
load balancer
service discovery
managed-provider endpoint
~~~

so applications do not need to know which physical machine is currently primary.

---

# 13. Stable Database Endpoint

Conceptual architecture:

~~~text
Application
    ↓
db.example.internal
    ↓
Proxy / Service Discovery
    ↓
Current Primary
~~~

During failover:

~~~text
Proxy target
Primary A
   ↓
Primary B
~~~

The application continues using the same logical endpoint.

---

# 14. DNS-Based Failover

DNS can redirect a database hostname to a new server.

However, DNS has considerations such as:

~~~text
TTL
client DNS caching
connection reuse
propagation delay
~~~

Existing TCP/database connections do not magically move to the new server.

Applications must handle connection failures and reconnect.

---

# 15. Proxy / Load-Balancer Approach

A database-aware or TCP proxy can expose a stable endpoint:

~~~text
Application
    ↓
Proxy
    ↓
Current Primary
~~~

After failover:

~~~text
Application
    ↓
same Proxy endpoint
    ↓
New Primary
~~~

Health checks and routing rules must correctly distinguish writable primary traffic from replica traffic.

---

# 16. Connection Pools During Failover

Your Node.js pool may contain connections to the failed primary.

~~~text
pg.Pool
  ├─ old connection ✗
  ├─ old connection ✗
  └─ old connection ✗
~~~

After failover, the application/driver must detect failures, discard unusable connections, and establish connections through the current database endpoint.

Connection pooling does not eliminate failover handling.

---

# 17. Application Retry

During failover, some requests may fail.

A robust application may retry selected operations, but retries must be designed carefully.

Example danger:

~~~text
Client sends INSERT
      ↓
connection breaks before response
      ↓
application does not know whether COMMIT happened
      ↓
blind retry
      ↓
possible duplicate operation
~~~

This is an **ambiguous commit outcome** problem.

Use idempotency and business keys where operations may be safely retried.

---

# 18. Idempotency During Failover

For payment/order creation, use an idempotency identifier:

~~~text
request_id = abc123
~~~

Database constraint:

~~~sql
UNIQUE (request_id)
~~~

Then a retry can check/reuse the existing operation rather than blindly creating another order.

HA is partly a database problem and partly an application-design problem.

---

# 19. RTO in HA

From Lesson 39:

~~~text
RTO = Recovery Time Objective
~~~

For HA, RTO strongly affects failover design.

Example:

~~~text
Primary fails at 10:00:00
New primary available at 10:00:30

RTO ≈ 30 seconds
~~~

Lower RTO generally requires more automation, redundancy, testing, and cost.

---

# 20. RPO in HA

~~~text
RPO = Recovery Point Objective
~~~

With asynchronous replication:

~~~text
Primary commit
    ↓
client receives success
    ↓
primary fails before replica receives latest WAL
~~~

some recent acknowledged transactions can be lost after failover.

That is an RPO consideration.

Synchronous replication can reduce certain data-loss windows but introduces latency and availability tradeoffs.

---

# 21. RTO vs RPO

~~~text
RTO
→ How long can service be unavailable?

RPO
→ How much committed data can we afford to lose?
~~~

HA design should begin with business requirements for both.

Do not choose synchronous replication or automatic failover only because they sound more advanced.

---

# 22. Availability vs Consistency Tradeoff

Suppose the primary cannot communicate with its synchronous standby.

Depending on configuration, you may face a choice between:

~~~text
continue accepting writes
or
preserve stronger cross-node durability guarantees
~~~

Distributed systems cannot remove every tradeoff.

Your HA architecture must define what should happen during partial failures.

---

# 23. Quorum / Consensus — Conceptual

Automated HA often needs a reliable way to determine which node is allowed to be leader.

Distributed coordination systems may use consensus/quorum concepts.

~~~text
Node A ─┐
Node B ─┼→ distributed consensus / leader state
Node C ─┘
~~~

A majority/quorum can help prevent isolated nodes from independently declaring themselves authoritative.

You do not need to implement a consensus algorithm to understand PostgreSQL HA, but you should know why coordination exists.

---

# 24. Patroni — High-Level

Patroni is a commonly used open-source PostgreSQL HA orchestration system.

Conceptually:

~~~text
PostgreSQL Node A + Patroni
PostgreSQL Node B + Patroni
PostgreSQL Node C + Patroni
          ↓
Distributed Configuration Store
          ↓
leader coordination / failover
~~~

Patroni can coordinate PostgreSQL replication, leader election, promotion, and failover using a distributed configuration store.

Examples of supported coordination backends vary by Patroni version/deployment.

Always check current Patroni documentation when implementing it.

---

# 25. Patroni Does Not Replicate Data Itself

Important distinction:

~~~text
PostgreSQL streaming replication
→ copies database changes

Patroni
→ orchestrates PostgreSQL HA state/failover
~~~

Patroni manages PostgreSQL nodes; PostgreSQL performs the underlying replication.

---

# 26. etcd / Consul / Kubernetes — Conceptual

HA orchestrators need shared coordination state.

Architectures may use systems such as:

~~~text
etcd
Consul
Kubernetes API/control plane
other supported distributed configuration stores
~~~

The exact choice depends on deployment.

Do not memorize a specific product combination as the only PostgreSQL HA architecture.

---

# 27. Example Self-Managed HA Architecture

~~~text
                   ┌───────────────────┐
                   │ Coordination Store │
                   └─────────┬─────────┘
                             │
            ┌────────────────┼────────────────┐
            ↓                ↓                ↓
      PostgreSQL A      PostgreSQL B      PostgreSQL C
       + Patroni         + Patroni         + Patroni
            │
            └──── current primary
                     ↓
               Proxy / HAProxy
                     ↓
                Application
~~~

Components solve different problems:

~~~text
PostgreSQL
→ data + replication

Patroni
→ HA orchestration

coordination store
→ leader/cluster state

proxy/service discovery
→ route clients
~~~

---

# 28. HAProxy — High-Level

HAProxy or similar proxies can route application connections based on node health/role.

Conceptually:

~~~text
Application
    ↓
HAProxy
   ├→ writable primary endpoint
   └→ read replicas endpoint
~~~

Health checks can integrate with an HA orchestrator/API to determine node roles.

Again, the exact stack is an implementation choice, not a PostgreSQL requirement.

---

# 29. Keepalived / Virtual IP — High-Level

Some architectures use a virtual IP that moves between nodes/proxies.

~~~text
Application
    ↓
Virtual IP
    ↓
Active DB/proxy endpoint
~~~

This can provide a stable network address.

Cloud and Kubernetes environments often use different service-discovery/load-balancing mechanisms instead.

---

# 30. Multi-AZ Architecture

Availability Zones are separate infrastructure failure domains within a region in many cloud platforms.

Conceptual architecture:

~~~text
AZ-A
Primary PostgreSQL
      │
      │ replication
      ↓
AZ-B
Standby PostgreSQL
~~~

If AZ-A has an infrastructure failure, the standby in AZ-B may be promoted.

Placing both nodes on the same physical failure domain weakens HA.

---

# 31. Why Failure Domains Matter

Bad:

~~~text
Primary + Standby
on same host/disk/power failure domain
~~~

One failure may destroy both.

Better:

~~~text
Primary → Failure Domain A
Standby → Failure Domain B
Backups → independent durable storage
~~~

HA is about independent failure domains, not merely the number of database processes.

---

# 32. Multi-Region — High-Level

For stronger geographic disaster resilience:

~~~text
Region A
Primary
   ↓ replication
Region B
Standby / DR replica
~~~

Tradeoffs increase:

~~~text
network latency
replication lag
cost
failover complexity
data residency
cross-region bandwidth
~~~

Synchronous cross-region replication may significantly increase write latency.

Use it only when business requirements justify it.

---

# 33. Managed PostgreSQL HA

Managed providers often offer HA features such as:

~~~text
standby replicas
automatic failover
managed health checks
stable endpoints
automatic backups
PITR
multi-zone placement
~~~

This removes much operational work, but not all responsibility.

You still need to understand:

~~~text
RTO/RPO guarantees
failover behavior
connection retry behavior
backup retention
maintenance events
provider limits
cost
~~~

---

# 34. Managed HA Does Not Mean Zero Downtime

During failover:

~~~text
existing DB connections may break
in-flight transactions may fail
new primary may take time to become available
application retries may be needed
~~~

Even managed HA should be tested from the application's perspective.

---

# 35. Automatic vs Manual Failover

Automatic failover:

~~~text
failure detected
   ↓
system promotes standby automatically
~~~

Advantages:

~~~text
lower RTO
less human delay
~~~

Risks:

~~~text
incorrect failure detection
unexpected promotion
split-brain risk if fencing/coordination is weak
~~~

Manual failover:

~~~text
operator validates failure
   ↓
operator initiates promotion
~~~

Slower, but can be appropriate for systems where false failover is extremely costly.

---

# 36. Planned Switchover Testing

A healthy HA system should be tested before an emergency.

Example:

~~~text
Primary A
   ↓ planned switchover
Standby B promoted
   ↓
Application reconnects
   ↓
A becomes/rebuilds as standby
~~~

This validates:

~~~text
replication
promotion
routing
application reconnection
monitoring
runbooks
~~~

---

# 37. Restore Redundancy After Failover

After a standby becomes primary:

~~~text
Before
Primary A → Standby B

After failure
A ✗
B = New Primary
~~~

You may now have no healthy standby.

HA work is not finished.

You must restore redundancy:

~~~text
New Primary B
      ↓
New/Rebuilt Standby C
~~~

---

# 38. Rejoining the Old Primary

Do not simply restart the old primary and let it accept writes.

Its timeline/history may have diverged.

Depending on circumstances, it may need:

~~~text
rewind/re-synchronization
rebuild from base backup
reconfiguration as standby
~~~

PostgreSQL provides tools such as `pg_rewind` for suitable scenarios, but safe use depends on prerequisites and failure history.

---

# 39. PostgreSQL Timelines — High-Level

When a standby is promoted, PostgreSQL creates a new WAL timeline.

Conceptually:

~~~text
Timeline 1
Primary A ──────────────X failure
                     \
                      \ promotion
                       ↓
Timeline 2             Primary B ─────────→
~~~

Timelines help PostgreSQL distinguish different WAL histories after recovery/promotion.

This is one reason failover is not simply "turn both servers back on."

---

# 40. HA and Backups

Even with excellent HA:

~~~text
Primary
   ↓
Standby
~~~

you still need:

~~~text
Backups
PITR
retention
restore testing
~~~

Why?

~~~text
accidental DELETE
bad migration
application bug
logical corruption
security incident
~~~

can affect replicated nodes.

---

# 41. HA and Connection Pooling

Architecture:

~~~text
Application instances
      ↓
connection pools
      ↓
stable HA endpoint / proxy
      ↓
current primary
~~~

After failover:

~~~text
old pooled connections fail
      ↓
pool removes/replaces connections
      ↓
new connections resolve through HA endpoint
      ↓
new primary
~~~

Configure sensible connection/acquisition/query timeouts so requests do not hang indefinitely.

---

# 42. HA and Transactions

If the primary fails during a transaction:

~~~text
BEGIN
 ↓
UPDATE
 ↓
primary failure
 ↓
connection lost
~~~

The application may receive an error and must determine how to respond.

Never assume a transaction succeeded simply because some statements ran before the connection failed.

Use business-level idempotency for safely retryable operations.

---

# 43. HA and Synchronous Replication

Possible design:

~~~text
Primary
   ↓ synchronous WAL
Standby A
   ↓
commit acknowledgement policy
~~~

This can improve RPO but increases dependency on network/standby health.

More complex configurations can select one or more synchronous standbys.

Design based on the required balance of:

~~~text
durability
latency
availability
cost
~~~

---

# 44. Quorum Synchronous Replication — High-Level

PostgreSQL can support synchronous policies where acknowledgements are required from a configured number of candidate standbys.

Conceptually:

~~~text
Primary
  ├→ Standby A ✓
  ├→ Standby B ✓
  └→ Standby C

Policy: wait for required number of acknowledgements
~~~

This can avoid depending on one specifically named standby while still requiring synchronous confirmation.

Exact syntax/configuration should be checked against the PostgreSQL version in use.

---

# 45. Monitoring HA

Monitor:

~~~text
primary health
standby health
replication lag
WAL retention
replication slots
database connections
disk space
CPU/memory/I/O
failover events
timeline changes
backup status
proxy/service health
~~~

HA without monitoring becomes failure detection by customer complaint.

---

# 46. Alerting

Useful alerts include:

~~~text
primary unreachable
replica disconnected
replication lag above threshold
disk nearly full
inactive slot retaining excessive WAL
backup failure
too many connections
failover occurred
no healthy standby
~~~

Alerts should be actionable rather than merely noisy.

---

# 47. Health Checks

A useful database health check should answer the right question.

Examples:

~~~text
Is PostgreSQL accepting connections?
Can a simple SQL query run?
Is this node primary or standby?
Is replication healthy?
Is the node too far behind?
~~~

A node can be alive but unsuitable for primary traffic.

---

# 48. Role Detection

PostgreSQL can indicate whether a server is in recovery:

~~~sql
SELECT pg_is_in_recovery();
~~~

High-level:

~~~text
false
→ normally primary / not in recovery

true
→ standby / recovery mode
~~~

HA systems/proxies can use role-aware health checks.

---

# 49. Chaos / Failure Testing

Do not wait for production failure to discover whether HA works.

Controlled testing can simulate:

~~~text
primary shutdown
network interruption
standby loss
proxy failure
application reconnection
replication lag
~~~

Then measure:

~~~text
Did failover happen?
How long did it take?
Was data lost?
Did the application recover?
Did alerts fire?
Was split brain prevented?
~~~

Perform such tests only in appropriately controlled environments.

---

# 50. HA Runbook

Document:

~~~text
How to identify current primary
How to inspect replicas
How to perform switchover
How to perform emergency failover
How fencing works
How applications reroute
How to validate new primary
How to rebuild a failed node
How to restore redundancy
How to roll back/escalate
~~~

During an incident, a tested runbook is more valuable than relying on memory.

---

# 51. ShopHub HA Example

Suppose ShopHub becomes revenue-critical.

Possible architecture:

~~~text
Users
  ↓
Next.js / API instances
  ↓
Stable DB endpoint / proxy
  ↓
Primary PostgreSQL — AZ-A
  │
  ├─ synchronous/asynchronous standby — AZ-B
  └─ backup + WAL/PITR storage
~~~

Failure:

~~~text
Primary AZ-A fails
      ↓
HA system confirms failure
      ↓
fences old primary
      ↓
promotes healthy AZ-B standby
      ↓
stable endpoint routes to new primary
      ↓
application pools reconnect
      ↓
new standby created
~~~

That is a complete HA story—not merely "we have replication."

---

# 52. CareerLoop Example

For CareerLoop today, a managed PostgreSQL provider may be the practical approach.

Instead of operating:

~~~text
Patroni
etcd
HAProxy
multiple PostgreSQL VMs
backup infrastructure
~~~

you can use provider-managed capabilities appropriate to the plan.

Your responsibility remains to understand:

~~~text
what HA the plan actually provides
backup/PITR retention
connection strategy
provider failover behavior
application retry/idempotency
RTO/RPO
~~~

Do not claim HA unless the deployed architecture actually provides it.

---

# 53. When Do You Need HA?

Ask:

~~~text
What does one hour of downtime cost?
How much data loss is acceptable?
How quickly must service recover?
Can operations handle self-managed HA?
What is the budget?
~~~

For a portfolio project:

~~~text
single managed PostgreSQL instance
+
backups
~~~

may be enough.

For a payment-critical production platform:

~~~text
multi-zone HA
+
tested failover
+
PITR/backups
~~~

may be justified.

---

# 54. HA Has a Cost

HA increases:

~~~text
infrastructure cost
operational complexity
monitoring requirements
testing requirements
network usage
engineering responsibility
~~~

The goal is not maximum complexity.

The goal is the simplest architecture that meets required availability and recovery objectives.

---

# Common Mistakes

## 55. Saying "We Have a Replica, So We Have HA"

A replica is only one HA component.

## 56. Promoting a Standby Without Fencing the Old Primary

This can create split brain.

## 57. Hard-Coding the Primary Server Address

Use a stable routing/service-discovery mechanism where HA requires transparent role changes.

## 58. Assuming Existing Connections Move During Failover

Connections usually break and applications must reconnect.

## 59. Blindly Retrying Every Failed Transaction

Ambiguous commit outcomes can create duplicate business operations.

## 60. Ignoring RPO While Optimizing RTO

Fast failover is not enough if unacceptable data is lost.

## 61. Assuming Synchronous Replication Has No Tradeoff

It can increase write latency and affect availability during standby/network problems.

## 62. Forgetting to Restore Redundancy

After failover, create/rebuild a healthy standby.

## 63. Treating HA as Backup

HA handles availability; backup/PITR handles historical recovery.

## 64. Never Testing Failover

An untested HA architecture is an assumption.

---

# Interview Revision

## What is PostgreSQL High Availability?

An architecture that minimizes database downtime by combining redundancy/replication with failure detection, safe failover, standby promotion, client rerouting, fencing, monitoring, and recovery procedures.

## Is replication the same as HA?

No. Replication provides another copy of changes. HA additionally requires safe failure detection, promotion, routing, fencing, and operational recovery.

## What is failover?

An unplanned switch from a failed/unhealthy primary to a standby that becomes the new primary.

## What is switchover?

A planned role transition between healthy database nodes, commonly for maintenance/testing.

## What is split brain?

A condition where multiple nodes accept writes as primary, causing divergent database histories.

## What is fencing?

Preventing an old/failed primary from continuing to accept conflicting writes when another node is promoted.

## Why use a stable DB endpoint?

It lets applications connect through a logical address/proxy/service rather than knowing which physical node is currently primary.

## What happens to connection pools during failover?

Connections to the old primary can fail. The application must discard/reconnect through the current HA endpoint.

## Why are retries difficult during failover?

If a connection breaks around COMMIT, the client may not know whether the transaction committed. Blind retries can duplicate operations.

## How can applications handle ambiguous retries?

Use idempotency keys, unique business constraints, and operation-specific retry logic.

## What is Patroni?

An HA orchestration system commonly used to coordinate PostgreSQL nodes, leader state, promotion, and failover. PostgreSQL streaming replication still performs the underlying data replication.

## RTO vs RPO?

RTO is acceptable recovery time; RPO is acceptable data loss.

## Why is HA not a backup?

Replicated nodes can receive the same accidental delete or bad migration. Backups/PITR preserve historical recovery options.

---

# Quick Revision

~~~text
POSTGRESQL HA

Application
    ↓
Stable endpoint / proxy
    ↓
Primary
    ↓ WAL
Standby
~~~

~~~text
FAILURE FLOW

Primary fails
     ↓
Detect failure
     ↓
Confirm / elect
     ↓
Fence old primary
     ↓
Promote standby
     ↓
Redirect clients
     ↓
Reconnect / retry safely
     ↓
Restore redundancy
~~~

~~~text
HA BUILDING BLOCKS

Replication
   +
Failure detection
   +
Failover orchestration
   +
Fencing
   +
Client routing
   +
Monitoring
   +
Backups/PITR
~~~

### Golden Distinction

~~~text
Replication
→ copies changes

High Availability
→ keeps service available through failures

Backup / PITR
→ restores historical state
~~~

### Golden Rule

~~~text
A standby is not enough.
Safe failover requires coordination + fencing + routing + recovery.
~~~

---

## Key Takeaway

> **PostgreSQL High Availability is not simply primary-plus-replica. A production HA design must detect failures correctly, prevent split brain through coordination and fencing, promote an appropriate standby, redirect applications through a stable endpoint, handle broken connections and ambiguous retries safely, and restore redundancy afterward. Choose synchronous/asynchronous replication, automation, and failure domains from explicit RPO/RTO requirements—and keep backups/PITR even when HA is excellent.**

---

[← Previous: Lesson 40 — PostgreSQL Replication](./40-replication.md) | [Back to Roadmap](../README.md) | [Next: Lesson 42 — Monitoring & Logging →](./42-monitoring-and-logging.md)
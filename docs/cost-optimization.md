# AWS Cost Optimization

Infrastructure cost was treated as an architectural constraint rather than something to optimize only after deployment. The useful unit is total cost: cloud spend, provider spend, engineering time, operational risk, and the cost of a slow or unreliable customer path.

Cost impact is expressed qualitatively because no controlled before-and-after baseline or complete billing export is available. Each conclusion is tied to an implemented access pattern.

## Case 1: persistent compute for the cohesive request core

### Problem

Customer, catalogue, payment, support, administrator, and affiliate routes are short interactive requests that share PHP code, PostgreSQL transactions, Redis identity state, and provider clients.

### Naive or alternative approach

Convert every route into a Lambda function because request-priced compute appears cheaper at low volume.

### Chosen design

Run the modular monolith on private EC2 behind WAF and an ALB. Use Lambda only for workloads with a distinct trigger or permission boundary.

### Cost impact

EC2 creates a fixed compute floor, but one deployment and runtime serve the entire request mix. This avoids per-function packaging, duplicated observability, connection-management work, and premature service boundaries. Persistent database connections and warm application code also avoid some repeated initialization.

### Performance impact

Interactive latency is predictable and does not include function cold starts. The trade-off is finite instance capacity rather than per-request elasticity.

### Trade-offs

The application scales as a coupled unit. A larger traffic profile may justify autoscaling or isolating a CPU-heavy module, but only after measurement.

## Case 2: PostgreSQL search instead of an OpenSearch domain

### Problem

Customers search across several catalogue entity types with weighted full-text matching, typo tolerance, and deterministic paging.

### Naive or alternative approach

Operate a managed search cluster as soon as product search exists.

### Chosen design

Use PostgreSQL full-text vectors, trigram similarity, bounded fallbacks, and indexed relational data. An OpenSearch library is packaged but no active application calls use it.

### Cost impact

The design reuses existing RDS capacity and avoids a separate always-on cluster, network path, index pipeline, monitoring surface, and data-synchronization process.

### Performance impact

For a bounded catalogue, queries remain close to the authoritative data and avoid cross-system freshness delay.

### Trade-offs

Search competes with transactional work for PostgreSQL CPU and I/O. A dedicated engine becomes justified when measured query concurrency, catalogue size, or relevance requirements exceed the relational budget.

## Case 3: S3/JSON as a browse projection

### Problem

Normal catalogue navigation repeatedly traverses mostly static topic, module, episode, and author structures.

### Naive or alternative approach

Reconstruct every browse page through relational joins on every request.

### Chosen design

Publish a hierarchical JSON projection to S3 and synchronize it into the active release. Keep PostgreSQL for search, entitlement, favorites, and purchase state.

### Cost impact

S3 is inexpensive durable publication storage, and local file reads remove repeated query load from RDS. One generated projection is reused across many anonymous/customer reads.

### Performance impact

Browse latency avoids database round trips and multi-table reconstruction.

### Trade-offs

The projection can diverge temporarily from PostgreSQL. Publication validation and versioning are needed to prevent a malformed or partial batch from affecting the catalogue.

## Case 4: Redis for expiring coordination and spend protection

### Problem

Session revocation, OTP attempts, resend cooldowns, checkout tokens, and daily messaging budgets are high-frequency states with natural expiration.

### Naive or alternative approach

Store every attempt and token in PostgreSQL, clean rows periodically, and let messaging-provider usage scale with incoming requests.

### Chosen design

Use Redis TTL keys, counters, and membership operations. The OTP guard combines identity/network limits with channel ceilings, daily provider-cost budgets, and alert throttling.

### Cost impact

Redis absorbs hot writes without growing relational tables or requiring cleanup jobs. More importantly, provider budgets place an architectural ceiling on abusive OTP spend instead of relying only on billing alerts after the cost occurs.

### Performance impact

Atomic in-memory operations keep security checks off the relational critical path.

### Trade-offs

Redis is a fixed managed-service cost and a security dependency, not merely a disposable cache. Availability, eviction policy, and memory sizing need explicit monitoring.

## Case 5: DynamoDB for irregular event records

### Problem

Consent evidence, login audit, suppression records, feedback, waiting-list entries, and privacy outcomes do not share the transaction model or retention period of the commercial database.

### Naive or alternative approach

Add every event class to PostgreSQL and scale the relational system for unrelated append/key workloads.

### Chosen design

Use DynamoDB for keyed event/audit records and TTL-based retention where appropriate. Lambda can access the same records without coupling to the PHP process.

### Cost impact

Event volume and retention scale separately from RDS. TTL removes expired records without an application cleanup worker. The actual capacity mode is not defined in this repository, so no request-price claim is made.

### Performance impact

Direct key operations provide predictable access for audit and workflow records.

### Trade-offs

Cross-store writes are not relationally atomic. A mailing-provider unsubscribe path uses a scan, whose read cost grows linearly; it should become an indexed lookup before table size makes that material.

## Case 6: direct CDN HLS delivery

### Problem

Video traffic contains long-lived connections and many segment requests. Serving it through PHP would make application scale follow media bandwidth.

### Naive or alternative approach

Proxy playlists and every segment through EC2 after checking entitlement.

### Chosen design

Perform one backend entitlement decision, mint short-lived CDN access, then let the player fetch adaptive HLS directly.

### Cost impact

EC2 worker count, application network capacity, and request logging do not grow with each media segment. Cached objects can serve multiple viewers through delivery infrastructure built for that workload.

### Performance impact

Users receive edge delivery and adaptive renditions; API capacity remains available for interactive work.

### Trade-offs

CDN transfer is still a variable cost. Signing policy, cache behavior, and origin configuration become operational dependencies and should be measured separately.

## Case 7: selective Lambda use

### Problem

Account deletion is infrequent but coordinates PostgreSQL, Redis, DynamoDB, secrets, payment cleanup, and a privacy audit. Other publication and reporting jobs are triggered by content or time rather than customers.

### Naive or alternative approach

Keep permanent worker capacity for rare jobs, or run long multi-system cleanup entirely inside a normal controller.

### Chosen design

Use Lambda for the isolated deletion workflow and production support workloads with event/schedule triggers, while retaining the interactive core on EC2.

### Cost impact

Rare work does not require an always-running dedicated worker fleet. Each function can have a narrow IAM boundary and scale independently from customer requests.

### Performance impact

Deletion receives an isolated execution budget. Scheduled aggregation removes repeated report computation from dashboard requests.

### Trade-offs

The synchronous deletion caller waits for completion, and cross-store partial failure still needs retry/reconciliation. Serverless execution does not create transactionality.

## Case 8: versioned deployment with bounded retention

### Problem

In-place deployments can leave mixed code/configuration and make recovery slow. Unlimited release retention consumes instance storage.

### Naive or alternative approach

Overwrite the live directory and keep every historical artifact.

### Chosen design

CodeDeploy prepares a versioned release, fetches configuration and S3 projections, switches an atomic symlink, and retains a small bounded set of recent releases.

### Cost impact

The design provides low-complexity release switching without a second permanent application fleet. Bounded retention prevents unbounded disk growth.

### Performance impact

Apache reloads gracefully and a local health poll validates the promoted release.

### Trade-offs

This is not traffic-shifted blue/green deployment. The hooks must fail closed on environment selection, and mutable package installation during deploy should move into a prebuilt image.

## Case 9: managed commercial capabilities

### Problem

Payment collection, subscription account management, invoicing, and high-deliverability email each require specialized compliance and operations.

### Naive or alternative approach

Build and operate equivalent payment UI, billing lifecycle, mail delivery, and invoice infrastructure inside the platform.

### Chosen design

Use hosted providers and keep the application focused on signed events, local entitlement, reconciliation, and user experience.

### Cost impact

This replaces engineering and operational burden with variable provider fees. It is economically rational while transaction volume and differentiation do not justify owning the infrastructure.

### Performance impact

Provider-hosted flows are mature and remove sensitive card handling from the application.

### Trade-offs

Provider outages, API changes, fees, and reconciliation complexity become explicit dependencies.

## Fixed, variable, and operational cost profile

| Cost type | Main components | Architectural control |
|---|---|---|
| Fixed | EC2, ALB, NAT, RDS, Redis | Small cohesive runtime; avoid unused search/worker clusters |
| Request/usage based | DynamoDB, Lambda, S3, CDN, providers | Workload-specific access, TTL, direct delivery, budgets |
| Operational | Deployments, incidents, reconciliation, security | Modular monolith, managed providers, atomic release process |

Private networking and NAT add fixed cost, but that cost buys a smaller public attack surface for application and data services. The decision is a security/economics trade rather than an attempt to minimize the bill at any cost.

## Next optimization work

The highest-value next steps are measurement-driven:

1. attribute RDS CPU between search and transactional traffic;
2. measure Redis memory by session, OTP, and checkout namespace;
3. replace DynamoDB scans before growth makes them expensive;
4. track CDN cache ratio, media egress, and authorization failures;
5. monitor NAT traffic by destination and consider private AWS service paths where justified;
6. reconcile payment and entitlement state automatically;
7. remove unused dependencies and generated coverage artifacts from release packages.

[Back to README](../README.md)

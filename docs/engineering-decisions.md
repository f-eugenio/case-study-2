# Engineering Decisions

## Decision: keep the interactive core as a modular monolith

### Context

Identity, catalogue, payments, entitlement, affiliate accounting, and support frequently share relational state.

### Problem

The platform needed a production boundary small enough for one technical owner without losing transaction integrity.

### Alternatives

- service-per-domain architecture;
- API Gateway and Lambda for every route;
- one persistent application with internal modules.

### Decision

Deploy one host-aware PHP application on private EC2 and isolate modules through route registries, controllers, models, and authorization policy.

### Why

One release and relational boundary reduce operational overhead and make payment/entitlement transitions easier to reason about.

### Trade-offs

Modules scale and deploy together. Route policy and shared-code discipline are critical.

### Conditions for reconsideration

Extract only a workload with independent scaling, ownership, or failure needs, such as webhook processing after measured bursts.

## Decision: use serverless selectively

### Context

Most traffic is synchronous, but account deletion and scheduled/publication workloads have different triggers and permissions.

### Problem

Rare multi-system work should not occupy a permanent worker or broaden normal application permissions.

### Alternatives

- run all work in EC2 controllers;
- operate a permanent worker fleet;
- move the full backend to functions;
- isolate only event-shaped workloads.

### Decision

Keep interactive HTTP on EC2 and use Lambda for account deletion plus separately deployed publication, synchronization, reporting, and audit workloads.

### Why

This aligns compute and IAM boundaries with the workload instead of applying one compute model universally.

### Trade-offs

The synchronous deletion call can approach function timeout, and multi-store operations still need partial-failure handling.

### Conditions for reconsideration

Persist deletion commands and execute a resumable state machine when completion latency or retry volume warrants it.

## Decision: retain search in PostgreSQL

### Context

Catalogue search requires full-text parsing, typo tolerance, relevance ordering, and stable pagination.

### Problem

A separate search domain adds fixed cost and index synchronization.

### Alternatives

- application-side filtering;
- managed OpenSearch;
- PostgreSQL full-text and trigram search.

### Decision

Use weighted PostgreSQL search across catalogue entity types. Treat the packaged OpenSearch client as unused until a measured need exists.

### Why

The bounded catalogue fits the relational capacity already being paid for and remains transactionally close to source data.

### Trade-offs

Search load competes with billing and entitlement queries.

### Conditions for reconsideration

Move only the search projection when database CPU, query latency, catalogue size, or relevance requirements exceed a defined threshold.

## Decision: use specialized stores by access pattern

### Context

The platform has relational commerce, expiring security state, event/audit records, and published objects.

### Problem

One database would either accumulate unsuitable workloads or require application-managed cleanup and contention controls.

### Alternatives

- PostgreSQL for every record;
- a universal document database;
- narrow use of PostgreSQL, Redis, DynamoDB, and S3.

### Decision

Use PostgreSQL for authoritative transactions, Redis for expiring coordination, DynamoDB for keyed audit/event records, and S3 for published projections.

### Why

Each store maps to concrete operations: transaction, TTL/counter, key/event, or object publication.

### Trade-offs

Cross-store transitions are not atomic, and each dependency adds monitoring and IAM surface.

### Conditions for reconsideration

Consolidate a store if its workload remains too small to justify the operational boundary, or add an outbox/reconciliation process if cross-store failure becomes frequent.

## Decision: publish a JSON browse projection

### Context

Catalogue browsing traverses mostly static hierarchy, while search and entitlement need dynamic data.

### Problem

Rebuilding the hierarchy from relational joins on every page wastes database capacity.

### Alternatives

- query PostgreSQL for every browse request;
- introduce a dedicated catalogue API/cache;
- publish JSON objects to S3 and synchronize them into releases.

### Decision

Serve browse metadata from a published JSON projection and keep PostgreSQL for search and user-specific state.

### Why

Local reads are simple and inexpensive, and S3 provides a durable publication handoff.

### Trade-offs

The read model can temporarily differ from PostgreSQL and needs schema/version validation.

### Conditions for reconsideration

Move the projection behind a CDN/object endpoint or dedicated catalogue service when publication frequency or fleet size makes per-instance synchronization awkward.

## Decision: make signed provider events authoritative

### Context

Checkout browser navigation can be interrupted or replayed, while providers retry webhooks.

### Problem

Local entitlement must change exactly once for a verified commercial event.

### Alternatives

- trust the return page;
- poll provider state on every media request;
- process signed events into local transactional state.

### Decision

Verify webhook signatures, claim events idempotently, and update PostgreSQL entitlement from provider events.

### Why

The design survives missing browser returns and duplicate provider delivery while keeping playback authorization local and fast.

### Trade-offs

Webhook handling becomes critical infrastructure and secondary provider effects require reconciliation.

### Conditions for reconsideration

Use a durable inbox/queue and independent workers when burst isolation, replay tooling, or multiple consumers justify them.

## Decision: combine JWT sessions with Redis revocation

### Context

Paid accounts require logout, device removal, security changes, and account deletion to invalidate access promptly.

### Problem

Purely stateless credentials cannot be revoked immediately without additional state.

### Alternatives

- long-lived server sessions in PostgreSQL;
- fully stateless JWT sessions;
- short-lived JWT sessions plus shared Redis state.

### Decision

Use short-lived JWT access sessions, refresh rotation, and server-side Redis validation/revocation.

### Why

The application retains compact credentials while gaining a shared decision point for immediate invalidation.

### Trade-offs

Redis is on the authentication critical path and must fail closed.

### Conditions for reconsideration

Partition revocation/session state and formalize availability objectives if authentication traffic grows substantially.

## Decision: keep EC2 out of the media transfer path

### Context

HLS playback produces many large, long-lived requests after one authorization decision.

### Problem

Proxying video through PHP would couple application capacity to viewing time and bitrate.

### Alternatives

- application proxy;
- permanent public media URLs;
- short-lived signed CDN access after entitlement validation.

### Decision

Authorize in PHP and deliver playlists/segments directly through the CDN.

### Why

It preserves server-side policy while using the correct infrastructure for cached media transfer.

### Trade-offs

CDN signing and origin configuration become security dependencies.

### Conditions for reconsideration

Add stronger content-protection controls only if licensing requirements and threat model justify the complexity.

## Decision: deploy versioned releases through an atomic symlink

### Context

The EC2 application must receive code, encrypted environment configuration, and S3-backed catalogue/configuration data as one operational release.

### Problem

Overwriting live files can expose partial deployments and makes recovery unclear.

### Alternatives

- edit the active directory in place;
- maintain a permanent blue/green fleet;
- prepare a versioned directory and atomically switch.

### Decision

Use CodeDeploy hooks to prepare a release, fetch configuration, switch a symlink, synchronize data, reload Apache, and run a local health probe.

### Why

It provides a small, understandable deployment model with bounded release retention.

### Trade-offs

The deployment hooks do not automatically restore the previous release after a post-switch failure, and environment selection needs a fail-closed correction.

### Conditions for reconsideration

Adopt immutable images and traffic-shifted deployment when fleet size, zero-downtime requirements, or rollback objectives require them.

[Back to README](../README.md)

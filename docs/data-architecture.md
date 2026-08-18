# Data Architecture

## Why one database was not used for everything

The platform has several data shapes with different consistency, latency, retention, and cost requirements. PostgreSQL is the authoritative relational core, but forcing short-lived counters, TTL records, object projections, and serverless audit events into that same path would increase write load and coupling.

The resulting design uses each store for a narrow access pattern:

| Store | Responsibility | Typical access |
|---|---|---|
| PostgreSQL | Authoritative business state and search | Relational joins, transactions, indexed lookup, full-text query |
| Redis | Expiring shared coordination | Key lookup, atomic counters, TTL, add/remove membership |
| DynamoDB | Keyed event and audit records | Put/get/update/query, TTL expiry, occasional batch deletion |
| S3 | Published catalogue/configuration objects | Deployment-time copy and directory synchronization |
| Media CDN/object origin | HLS delivery artifacts | Playlist and segment retrieval through signed access |

## PostgreSQL: the transactional system of record

PostgreSQL owns data that must remain relationally coherent:

- accounts, credentials, devices, and account status;
- catalogue entities and relationships;
- subscriptions, transactions, one-time purchases, and entitlements;
- favorites and customer library state;
- affiliate attribution, commissions, and reporting state;
- support cases and conversation messages.

Payment activation is the clearest example. The payment notification, customer entitlement, purchase state, and affiliate commission can affect one another. Keeping these records in one relational boundary permits transactions, uniqueness constraints, and idempotent inserts.

### Connection behavior

The PDO connection wrapper adds production-oriented failure limits:

- encrypted database connections with certificate verification when the CA is available;
- native prepared statements;
- finite connection, statement, lock, and idle-transaction timeouts;
- bounded retry with exponential backoff and jitter for transient failover errors;
- an optional short circuit breaker through APCu.

Retries are limited to transient connection classes. Business errors are not retried blindly.

### Catalogue search

Search remains in PostgreSQL. The query implementation combines:

- language-aware full-text vectors;
- weighted matches across content fields;
- fuzzy similarity and bounded fallback matching;
- deterministic tie ordering;
- bounded pages and cursor-aware result windows.

This avoids an always-on search domain while the catalogue remains within relational-search capacity. The OpenSearch client is present in dependency metadata but no active OpenSearch request path exists.

## Redis: expiring security and workflow state

Redis is not used as a generic copy of PostgreSQL. Its active responsibilities are states that expire naturally or require cheap atomic coordination:

- access and refresh session validation and revocation;
- refresh-rotation grace state;
- OTP values, expiry, resend cooldown, and verification attempts;
- per-user, per-address, per-channel, and global abuse counters;
- daily provider-spend budgets and alert throttles;
- checkout and one-time-purchase tokens;
- short-lived legal-consent snapshots awaiting provider confirmation.

Most access is direct key lookup, increment, membership, and expiry. This prevents a relational write on every authenticated refresh or OTP attempt and makes automatic cleanup part of the data model.

The trade-off is criticality: when Redis is unavailable, authentication refresh, OTP, and checkout coordination cannot safely pretend that shared state still exists.

## DynamoDB: irregular keyed records

DynamoDB holds records that are event-shaped, independently retained, or useful to serverless workflows:

- legal-consent evidence;
- login and device audit events;
- mailing-list and waiting-list records;
- promotional-code and campaign records;
- email suppression and provider webhook events;
- user feedback;
- privacy and account-deletion audit records.

TTL is used where retention is time-bound. Longer-lived evidence is pseudonymized before storage where the implementation requires durable proof without retaining the original identifying value.

### Access-pattern trade-offs

Most paths use direct key operations or queries. One mailing-provider unsubscribe path scans for an address before updating matching items. That is functional for a small table but has linear read cost and latency. A purpose-built secondary index or normalized lookup key is the next step when volume makes the scan material.

DynamoDB does not share a transaction with PostgreSQL. Auxiliary audit writes are therefore either staged through Redis, made after the relational transition, or treated as best effort. This must be visible operationally rather than hidden behind a claim of atomicity.

## S3: published read models and release inputs

S3 stores environment-specific JSON objects for:

- the browse-oriented catalogue projection;
- offers and coupon configuration consumed by the release.

CodeDeploy hooks copy commercial configuration into the new release, switch the application symlink, and synchronize the active catalogue directory with exact timestamps and deletion of removed objects.

This is a lightweight read-model pattern. Browse pages can traverse static nested JSON without repeatedly joining catalogue tables. Search and entitlement still use PostgreSQL.

The trade-offs are:

- S3 and PostgreSQL can temporarily describe different publication states;
- release correctness depends on a successful synchronization;
- invalid objects can affect many reads at once.

Schema validation, publication versioning, and a last-known-good manifest are natural controls at a larger catalogue scale.

## Media data

The application consumes HLS renditions through a CDN. The repository confirms entitlement checks, URL signing, and player delivery. The media-generation infrastructure is outside the request core; application EC2 never becomes the segment store.

## Consistency model by workflow

### Checkout and activation

1. PostgreSQL records pending commercial state.
2. Redis stores short-lived checkout and consent context.
3. The payment provider emits a signed event.
4. PostgreSQL claims the event and updates authoritative entitlement transactionally.
5. DynamoDB persists consent evidence and external systems are synchronized.

A provider retry reaches the idempotency checks instead of duplicating entitlement or commission records.

### Account deletion

The Lambda spans PostgreSQL, Redis, DynamoDB, and providers. It cannot make those systems one transaction. It uses revocation first, targeted deletion/anonymization, best-effort cleanup for auxiliary records, and a final pseudonymous audit result.

### Catalogue publication

S3 is the published browse projection; PostgreSQL is the searchable and transactional catalogue. Synchronization is a deployment operation rather than a distributed transaction.

## Cost consequences

- PostgreSQL search reuses existing fixed relational capacity instead of adding a search cluster.
- Redis absorbs high-frequency expiring counters without growing relational tables.
- DynamoDB lets irregular event volume scale independently, but scan-based paths must remain small or be indexed.
- S3 serves as inexpensive durable publication storage and avoids repeated catalogue joins.
- Direct CDN delivery prevents application compute and network capacity from scaling with every media segment.

# Identity and Device Access

## Scope

Identity spans registration, email verification, password recovery, customer/administrator/affiliate entry points, device awareness, recent-access history, step-up verification, refresh rotation, and immediate server-side revocation.

The model has three layers:

1. prove control of an account or communication channel;
2. establish the application identity and device context;
3. continuously confirm that the session is still accepted.

See the [identity flow](../diagrams/identity-device-flow.md).

## Account lifecycle

### Registration and recovery

Registration validates account data, checks account uniqueness, creates an expiring email challenge, and inserts the relational account only after verification. Password recovery and sensitive profile changes reuse the step-up pattern.

Challenge generation uses a cryptographic random source. Redis holds only the active verification state and deletes it through expiry or successful completion.

### Device awareness

Login evaluates both credentials and a device context. A new or untrusted context can require additional verification. The account area exposes recent access records and lets the user remove a device.

Device and login events are written to DynamoDB as audit-shaped records, while the authoritative account and device relationship remains in PostgreSQL. Removing a device also invalidates the associated shared session state.

### Separate identity surfaces

Customer, administrator, and affiliate routes have separate login controllers, route registries, and Redis namespaces. Common mechanics are reused, but each surface has a narrower route set and distinct authorization expectations.

This limits accidental crossover. The trade-off is duplicated identity logic that can drift; common issuance, revocation, and audit primitives should remain centralized.

## Session lifecycle

The API uses short-lived JWT access sessions with refresh rotation and server-side validation/revocation in Redis. This is intentionally stateful where immediate control matters.

Redis is required for:

- logout and device removal;
- refresh rotation and limited grace handling;
- account-security changes;
- revocation before credential expiry;
- deletion of all active sessions during account removal.

A fully stateless approach would remove the Redis lookup but weaken immediate revocation. For a paid-content account, the explicit security dependency is the better trade-off.

## OTP as security and cost control

The OTP subsystem does more than compare a code. Redis coordinates:

- challenge TTL;
- resend cooldown schedules;
- failed verification attempts;
- per-user, per-address, and per-channel counters;
- global throughput ceilings;
- provider-spend budgets and throttled operational alerts.

This prevents abusive clients from converting a public recovery or signup endpoint into unbounded messaging-provider spend.

The controls are endpoint-specific and active in the email/identity workflows. A separate generic application rate-limiter helper exists but is not wired into the global request constructor; it should not be described as platform-wide rate limiting.

## Authorization

Authentication establishes who the caller is. Controllers still decide what that identity may do.

Confirmed authorization checks include:

- login-required route metadata;
- media entitlement based on local subscription or one-time purchase;
- support-ticket ownership;
- payment-customer ownership;
- operator identity derived from server-side session state;
- separate administrator and affiliate surfaces.

Authorization is not completely centralized. A route-policy test that enumerates every mutation, required identity, and resource rule would reduce drift and stale route risk.

## API-key boundary

The browser API key identifies an expected application client and can support traffic classification. It is not confidential and is not used as proof of user identity. Protected actions still require authenticated identity and controller-level authorization.

Provider webhooks do not use that browser key. They validate their own signatures before dispatch.

## Account deletion

Self-service deletion requires authentication and an OTP verification step. The controller rejects deletion while a subscription remains active, performs mailing-provider cleanup on a best-effort basis, and invokes the account-deletion Lambda.

The Lambda revokes Redis state before coordinating deletion or anonymization across PostgreSQL, DynamoDB, and the payment provider. It returns a structured status so partial failure is visible rather than represented as an atomic multi-store success.

[Account-deletion architecture →](event-driven-architecture.md)

## Failure posture and hardening

| Concern | Current behavior | Hardening direction |
|---|---|---|
| Redis unavailable | Shared verification/revocation cannot be confirmed | Fail closed and alert on dependency health |
| OTP abuse | Distributed TTL counters and budgets | Tune from observed traffic and provider costs |
| Route drift | Registry metadata plus controller checks | Generate authorization-matrix tests |
| CSRF | Applied on selected browser mutations | Centralize middleware for all state changes |
| Identity duplication | Separate customer/admin/affiliate flows | Share primitives without merging permissions |
| Login audit failure | DynamoDB write can fail independently | Measure and reconcile security-event loss |

[Back to README](../README.md)

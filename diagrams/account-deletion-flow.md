# Account-Deletion Flow

```mermaid
sequenceDiagram
    actor User
    participant App as PHP application
    participant L as Account-deletion Lambda
    participant Secrets as Secrets Manager
    participant R as Redis
    participant PG as PostgreSQL
    participant DDB as DynamoDB
    participant Providers as Payment / mailing providers

    User->>App: authenticated deletion request
    App->>App: verify OTP and subscription policy
    App->>L: synchronous scoped invocation
    L->>Secrets: load runtime configuration
    L->>R: revoke all active sessions
    L->>PG: delete or anonymize relational data
    L->>DDB: remove linked login/audit records
    L->>Providers: perform applicable provider cleanup
    L->>DDB: write pseudonymous outcome record
    L-->>App: complete, partial, or failed
    App-->>User: controlled result
```

The workflow is isolated because it is rare, permission-sensitive, and spans systems that cannot share a transaction. Revocation happens early; partial completion remains visible for investigation and retry.

[Back to README](../README.md)

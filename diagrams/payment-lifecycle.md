# Payment Lifecycle

```mermaid
sequenceDiagram
    actor User
    participant App as PHP application
    participant R as Redis
    participant PG as PostgreSQL
    participant Pay as Payment provider
    participant DDB as DynamoDB
    participant Side as Invoicing / mailing providers

    User->>App: prepare subscription or one-time purchase
    App->>PG: record pending transaction
    App->>R: store expiring checkout and consent state
    App->>Pay: create hosted flow with idempotency
    Pay-->>User: collect payment details
    Pay->>App: signed lifecycle webhook
    App->>PG: claim provider event
    alt new valid event
        App->>PG: transactionally update purchase and entitlement
        App->>R: consume staged state
        App->>DDB: persist consent evidence
        App->>Side: synchronize invoice and lifecycle data
        App-->>Pay: success
    else duplicate event
        App-->>Pay: idempotent success
    end
```

The signed provider event, not the browser return, controls local access. PostgreSQL is authoritative; Redis and DynamoDB support different retention and workflow needs around that transition.

[Back to README](../README.md)

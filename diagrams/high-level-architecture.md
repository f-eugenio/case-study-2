# High-Level Architecture

```mermaid
flowchart LR
    Customer["Customer browser"]
    Ops["Administrator / affiliate browser"]
    Payment["Payment provider"]
    Communications["Email, mailing, invoicing providers"]
    CDN["Media CDN"]

    subgraph Edge["Public AWS boundary"]
        WAF["AWS WAF"]
        ALB["Application Load Balancer"]
        NAT["NAT egress"]
    end

    subgraph Private["Private subnet tier across three AZs"]
        App["PHP modular monolith<br/>EC2"]
        PG[("RDS PostgreSQL<br/>transactions and search")]
        Redis[("ElastiCache Redis<br/>sessions and expiring state")]
    end

    DDB[("DynamoDB<br/>audit and event records")]
    S3[("S3<br/>published catalogue/config")]
    Delete["Account-deletion Lambda"]
    Deploy["CodeDeploy + Systems Manager"]

    Customer -->|"HTTPS"| WAF --> ALB --> App
    Ops --> WAF
    Payment -->|"signed webhooks"| WAF

    App --> PG
    App --> Redis
    App --> DDB
    App -->|"provider calls"| NAT --> Communications
    NAT --> Payment

    App -->|"temporary media access"| Customer
    Customer -->|"HLS playlists and segments"| CDN

    Deploy --> App
    S3 -->|"release synchronization"| App
    App -->|"verified request"| Delete
    Delete --> PG
    Delete --> Redis
    Delete --> DDB
```

The interactive application remains one cohesive deployment. Specialized services are used for concrete access patterns: relational transactions, expiring coordination, event-shaped records, object publication, isolated compliance work, and direct media delivery.

[Back to README](../README.md)

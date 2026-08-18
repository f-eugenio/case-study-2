# Network and Security Architecture

```mermaid
flowchart TB
    Internet["Internet clients"]
    Webhooks["Provider webhooks"]
    WAF["AWS WAF"]

    subgraph VPC["VPC across three availability zones"]
        subgraph Public["Public subnet tier"]
            PubA["Public subnet · AZ A"]
            PubB["Public subnet · AZ B"]
            PubC["Public subnet · AZ C"]
            ALB["Application Load Balancer<br/>spans the public subnet group"]
            NAT["NAT egress capacity<br/>placed in the public tier"]
        end

        subgraph Private["Private subnet tier"]
            PrivA["Private subnet · AZ A"]
            PrivB["Private subnet · AZ B"]
            PrivC["Private subnet · AZ C"]
            EC2["EC2 application capacity<br/>eligible private-subnet placement"]
            RDS["RDS service and subnet group<br/>private placement"]
            Redis["ElastiCache service and subnet group<br/>private placement"]
        end
    end

    SSM["Systems Manager<br/>administrative control plane"]
    Regional["Regional AWS services<br/>DynamoDB · S3 · Lambda · Secrets"]
    External["Billing, email, invoicing,<br/>mailing and media providers"]

    Internet -->|"HTTPS"| WAF
    Webhooks -->|"HTTPS"| WAF
    WAF --> ALB
    PubA --- ALB
    PubB --- ALB
    PubC --- ALB
    ALB -->|"application target group"| EC2

    PrivA --- EC2
    PrivB --- EC2
    PrivC --- EC2
    PrivA --- RDS
    PrivB --- RDS
    PrivC --- RDS
    PrivA --- Redis
    PrivB --- Redis
    PrivC --- Redis

    EC2 -->|"database security-group rule"| RDS
    EC2 -->|"cache security-group rule"| Redis
    SSM -->|"managed session"| EC2
    EC2 -->|"AWS API access"| Regional
    EC2 -->|"private route"| NAT
    NAT -->|"controlled egress"| External
```

## How to read this diagram

There are three public and three private subnets, distributed across availability zones. The ALB spans the public subnet group. Application compute and private managed data services use the private subnet group; NAT supplies outbound connectivity from private routes.

The connections to all three private subnets describe eligible placement, not one copy of each service in every subnet. EC2, RDS, ElastiCache, and NAT availability-zone placement is determined by deployment and managed-service configuration.

## Security boundaries

- WAF and the ALB are the public application boundary.
- EC2 has no direct client ingress path.
- PostgreSQL and Redis accept only the required application security-group relationships.
- Systems Manager provides administrative access without a public SSH route.
- NAT supplies controlled outbound connectivity for external providers.
- DynamoDB, S3, Lambda, and Secrets Manager are regional AWS services, not resources placed inside one application subnet.

[Back to README](../README.md)


# Request Routing

```mermaid
flowchart TD
    Request["HTTPS request"]
    Edge["WAF and ALB"]
    Front["PHP front controller"]
    Health{"Health probe?"}
    Host["Normalize host and origin"]
    Surface{"Select host surface"}

    Customer["Customer routes"]
    Admin["Administrator routes"]
    Affiliate["Affiliate routes"]
    Staging["Staging routes"]

    Api{"API route?"}
    Provider{"Provider webhook?"}
    ClientId["Client identification check"]
    Signature["Provider signature verification"]
    Policy["Route metadata<br/>auth · DB · navigation"]
    Controller["Controller authorization and action"]

    Request --> Edge --> Front --> Health
    Health -->|"yes"| Fast["Minimal health response"]
    Health -->|"no"| Host --> Surface
    Surface --> Customer
    Surface --> Admin
    Surface --> Affiliate
    Surface --> Staging
    Customer --> Api
    Admin --> Api
    Affiliate --> Api
    Staging --> Api
    Api -->|"normal API"| ClientId --> Policy
    Api -->|"payment / mailing webhook"| Provider --> Signature --> Policy
    Policy --> Controller
```

Host-aware registries allow several product surfaces to share one deployment. The browser API key identifies an expected client but is not an authorization boundary. Provider webhooks authenticate through their own signatures, and sensitive controllers apply ownership, role, or entitlement checks after routing.

[Back to README](../README.md)

```mermaid
flowchart LR
    subgraph Kubernetes Cluster
        F[Fetcher]
        B[Backend]
        R[Redis]
        D[Dex]
    end

    F -->|Data| R
    B -->|gRPC| R
    B -->|OIDC| D
    L[LDAP] --> D
```
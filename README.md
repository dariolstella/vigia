# Vigia

<p style="text-align: center"><img src="assets/vigia.png"></p>


# Plataforma de visualización de métricas

# Stack tecnologico

| Componente   | Tecnología                    | Protocolo      | Puerto | Función Principal                     |
|--------------|-------------------------------|----------------|--------|---------------------------------------|
| Fetcher      | Go, Resty, go-redis           | HTTP/Redis     | -      | Colector de métricas                  |
| Backend      | Go (Gin)/Node.js (Express)    | HTTPS          | 3001   | API Gateway + Transformación de datos |
| Frontend     | React, Cytoscape.js           | HTTPS          | 3000   | Visualización interactiva             |
| Redis        | Redis 7+                      | TLS            | 6379   | Caché temporal de métricas            |
| Dex          | Dex IDP                       | HTTPS/LDAPS    | 5556   | Proveedor de identidad OIDC           |
| LDAP         | Active Directory              | LDAPS          | 389    | Directorio de usuarios                |


# Arquitectura Completa del Sistema

```mermaid
%% Complete Architecture Diagram
flowchart TD
    subgraph External Services
        DYNA[Dynatrace API HTTPS:443] -->|Production| FETCHER
        MOCK[Mock Dynatrace HTTP:8080] -->|Development| FETCHER
    end

    subgraph Security Layer
        DEX[Dex IDP HTTPS:5556] -->|LDAPS| LDAP[(LDAP Server AD LDAPS:389)]
    end

    subgraph Data Layer
        FETCHER[Fetcher Service ENV: API_TOKEN] -->|SET metrics:data| REDIS[(Redis Cluster TLS:6379)]
    end

    subgraph Application Layer
        BACKEND[Backend API HTTPS:3001] -->|GET metrics:data| REDIS
        BACKEND -->|Token Validation| DEX
    end

    subgraph Client Layer
        FRONTEND[Frontend HTTPS:3000] -->|OIDC Flow| DEX
        FRONTEND -->|API Requests| BACKEND
        FRONTEND -->|Visualization| CYTO[Cytoscape Graph]
    end

    style DYNA fill:#74b9ff,stroke:#0984e3
    style MOCK fill:#a29bfe,stroke:#6c5ce7
    style DEX fill:#7b2cbf,stroke:#5a189a
    style LDAP fill:#4895ef,stroke:#4361ee
    style REDIS fill:#ff7675,stroke:#d63031
    style FETCHER fill:#55efc4,stroke:#00b894
    style BACKEND fill:#ffeaa7,stroke:#fdcb6e
    style FRONTEND fill:#81ecec,stroke:#00cec9
```


# Diagrama de integraciones
```mermaid
flowchart LR
    A[Fetcher] --> B[Redis]
    C[Frontend] --> D[Backend]
    D --> B
    D --> E[Dex]
    E --> F[LDAP]
    G[Prometheus] -->|Scrape| A
    G -->|Scrape| D
    H[Grafana] --> G
```


# Flujo de datos
```mermaid
sequenceDiagram
    participant D as Dynatrace
    participant F as Fetcher
    participant R as Redis
    participant B as Backend
    participant Fr as Frontend

    F->>D: GET /api/v2/metrics/query
    D-->>F: JSON Response
    F->>R: SETEX metrics_data 3600 <JSON>
    Fr->>B: GET /api/metrics (JWT)
    B->>R: GET metrics_data
    R-->>B: Raw Data
    B->>B: Transform to Graph Format
    B-->>Fr: Nodes/Edges JSON
    Fr->>Fr: Render Cytoscape
```


# Autenticación y autorización de componentes
```mermaid
sequenceDiagram
    participant U as Usuario
    participant Fr as Frontend
    participant D as Dex
    participant L as LDAP

    U->>Fr: Login Request
    Fr->>D: Redirect to /auth
    D->>L: LDAP Bind (cn=user)
    L-->>D: Success + Groups
    D->>Fr: Auth Code (302)
    Fr->>D: Exchange Code for Token
    D-->>Fr: Access Token (JWT)
    Fr->>Fr: Store Token (HttpOnly)
```


# Diagrama de componentes a escalar
```mermaid
flowchart LR
    subgraph Kubernetes Cluster
        F[Fetcher]
        B[Backend]
        D[Dex]
    end

    F -->|Data| R[Redis]
    B -->|gRPC| R[Redis]
    B -->|OIDC| D
    L[LDAP] --> D
```
# HyperFleet Gateway: Request Flow

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant Envoy as Envoy Gateway
    participant Authorino
    participant API as HyperFleet API
    participant DB as Database

    Client->>Envoy: Request with Auth Header (and possible forged headers)
    Note over Envoy: Early header mutation<br/>(Strips client-supplied x-tenant-* and x-hyperfleet-* headers)
    Envoy->>Authorino: ext_authz Check (over gRPC/TLS)
    
    rect rgb(240, 248, 255)
        Note over Authorino: Authentication Pipeline
        Authorino->>Authorino: Validates Credential (OIDC or TokenReview based on Scheme)
        Authorino->>Authorino: Maps Tenant Dimensions & Identity
        Authorino->>Authorino: Mints Wristband (Signed JWT)
    end
    
    Authorino-->>Envoy: Allow + Injected Headers (x-tenant-*, hf_system) + Wristband
    Envoy->>API: Forwards Request over TLS (Trusted Headers + Wristband)
    
    rect rgb(240, 255, 240)
        Note over API: In-app Validation
        API->>API: JWT Middleware (Validates exclusively the Wristband)
        API->>API: Tenant Middleware (Extracts scope via headers/claims)
    end
    
    API->>DB: Query with containment filter (tenancy @> caller_map)
    DB-->>API: Tenant-scoped data
    API-->>Client: Tenant-scoped response



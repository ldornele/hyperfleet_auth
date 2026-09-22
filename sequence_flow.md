# HyperFleet Gateway: Request Flow

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant Envoy as Envoy Gateway
    participant Authorino
    participant API as HyperFleet API
    participant DB as Database

    Client->>Envoy: Requisição com Auth Header (e possíveis cabeçalhos falsos)
    Note over Envoy: Early header mutation<br/>(Remove cabeçalhos x-tenant-* e x-hyperfleet-* do cliente)
    Envoy->>Authorino: ext_authz Check (via gRPC/TLS)
    
    rect rgb(240, 248, 255)
        Note over Authorino: Pipeline de Autenticação
        Authorino->>Authorino: Valida Credencial (OIDC ou TokenReview baseado no Scheme)
        Authorino->>Authorino: Mapeia Tenant Dimensions & Identity
        Authorino->>Authorino: Gera o Wristband (Signed JWT)
    end
    
    Authorino-->>Envoy: Allow + Injeção de Headers (x-tenant-*, hf_system) + Wristband
    Envoy->>API: Encaminha Requisição sobre TLS (Headers confiáveis + Wristband)
    
    rect rgb(240, 255, 240)
        Note over API: Validação in-app
        API->>API: JWT Middleware (Valida exclusivamente o Wristband)
        API->>API: Tenant Middleware (Extrai scope via headers/claims)
    end
    
    API->>DB: Query com filtro de contenção (tenancy @> caller_map)
    DB-->>API: Dados com escopo isolado
    API-->>Client: Resposta isolada por tenant

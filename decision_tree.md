# HyperFleet Gateway: Request Flow

Este diagrama ilustra o fluxo completo de uma requisição no modo `edge+api`, demonstrando a mutação de cabeçalhos, a validação pelo Authorino, a geração do *wristband* (JWT) e o isolamento de *tenancy* no banco de dados.

```mermaid
  graph TD
      Start([Request Intercepted by Envoy]) --> ExtractAuth[Extract Authorization Header]
      ExtractAuth --> CheckScheme{What is the Scheme?}
  
      %% Bearer Flow (Humans)
      CheckScheme -- "Bearer" --> OIDC[OIDC Method for Humans]
      OIDC --> ValidateJWT[Validate JWT against OIDC_ISSUER_URL]
      ValidateJWT --> CheckTenantClaim{Has tenant claim?}
      CheckTenantClaim -- Yes --> AllowHuman[Allow Request<br/>hf_system: false]
      CheckTenantClaim -- No --> DenyHuman[Deny: 403 Forbidden]
  
      %% ServiceAccount Flow (Machines)
      CheckScheme -- "ServiceAccount" --> SA[TokenReview Method for Machines]
      SA --> CallTR[Call Kubernetes TokenReview API<br/>Audience: hyperfleet-api]
      CallTR --> CheckAllowlist{Subject in allow-list?}
      CheckAllowlist -- Yes --> AllowMachine[Allow Request<br/>hf_system: true]
      CheckAllowlist -- No --> DenyMachine[Deny: 401 Unauthorized]
  
      %% Unknown Flow
      CheckScheme -- "Anything Else / Missing" --> DenyUnknown[Deny at Gateway: 401]

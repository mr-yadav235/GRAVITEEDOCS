# User Interaction & Authentication Flows

## Overview

This document details how different users interact with the Gravitee Platform and authenticate via OpenAM SSO (OAuth 2.0 / OIDC).

![User Interaction Flows](./diagrams/USER-INTERACTION-FLOWS.svg)

---

## User Types

| User Type | Zone | Access | Purpose |
|-----------|------|--------|---------|
| **Corporate User** | Corporate Zone | Console UI | API Lifecycle Management, Administration |
| **App Developer** | Corporate Zone | Developer Portal | Browse APIs, Subscribe, Get API Keys |
| **API Consumer App** | Corporate Zone | API Gateway | Call APIs with OAuth 2.0 tokens |

---

## Flow A: Corporate User → Console UI

### Overview

Corporate Users (API Admins, Platform Team) access the Console UI to manage API lifecycles.

```
Corporate Zone          Inner DMZ           Core Zone
┌─────────────┐      ┌───────────┐      ┌──────────────┐
│  Corporate  │──A1─►│  OpenAM   │◄─A2──│  Console UI  │
│    User     │◄─────│   SSO     │──────│              │
└─────────────┘      └───────────┘      └──────────────┘
       │                                        │
       └────────────────A3─────────────────────►│
                    (with JWT token)
```

### Flow Steps

| Step | Action | Details |
|------|--------|---------|
| **A1** | Access Console | User navigates to `https://console.internal.company.com` |
| **A2** | Auth Redirect | Console redirects to OpenAM `/oauth2/authorize` |
| **A3** | Login | User enters corporate credentials (validated against LDAP/AD) |
| **A4** | Token Exchange | Console receives authorization code, exchanges for JWT |
| **A5** | Session Created | User logged in, can manage APIs |

### Mermaid Diagram

```mermaid
sequenceDiagram
    autonumber
    participant User as 👤 Corporate User
    participant Console as 💻 Console UI
    participant OpenAM as 🔐 OpenAM SSO
    participant LDAP as 📁 LDAP/AD
    
    User->>Console: Access console.internal.company.com
    Console-->>User: 302 Redirect to OpenAM
    User->>OpenAM: Login page
    User->>OpenAM: Submit credentials
    OpenAM->>LDAP: Validate
    LDAP-->>OpenAM: OK ✅
    OpenAM-->>User: 302 Redirect with code
    User->>Console: /callback?code=xxx
    Console->>OpenAM: Exchange code for token
    OpenAM-->>Console: JWT token
    Console-->>User: Dashboard
```

---

## Flow B: App Developer → Developer Portal

### Overview

Application Developers access the Developer Portal to browse APIs, view documentation, and subscribe to get API credentials.

```
Corporate Zone          Inner DMZ           Core Zone
┌─────────────┐      ┌───────────┐      ┌──────────────┐
│    App      │──B1─►│  OpenAM   │◄─B2──│   Developer  │
│  Developer  │◄─────│   SSO     │──────│    Portal    │
└─────────────┘      └───────────┘      └──────────────┘
       │                                        │
       └────────────────B3─────────────────────►│
                  (Browse & Subscribe)
```

### Flow Steps

| Step | Action | Details |
|------|--------|---------|
| **B1** | Access Portal | Developer navigates to `https://portal.internal.company.com` |
| **B2** | Auth Redirect | Portal redirects to OpenAM for authentication |
| **B3** | Login | Developer enters corporate credentials |
| **B4** | Token Exchange | Portal receives JWT token |
| **B5** | Browse APIs | Developer views API catalog and documentation |
| **B6** | Subscribe | Developer subscribes to API, receives API Key or OAuth client credentials |

### Mermaid Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Dev as 👨‍💻 App Developer
    participant Portal as 📚 Developer Portal
    participant OpenAM as 🔐 OpenAM SSO
    participant API as 📊 Mgmt API
    
    Dev->>Portal: Access portal.internal.company.com
    Portal-->>Dev: 302 Redirect to OpenAM
    Dev->>OpenAM: Login
    OpenAM-->>Dev: 302 Redirect with code
    Dev->>Portal: /callback?code=xxx
    Portal->>OpenAM: Exchange for token
    OpenAM-->>Portal: JWT token
    Dev->>Portal: Browse APIs
    Portal->>API: GET /portal/apis
    API-->>Portal: API list
    Portal-->>Dev: Display catalog
    Dev->>Portal: Subscribe to API
    Portal->>API: POST /subscriptions
    API-->>Portal: API Key: ak_xxx
    Portal-->>Dev: Show credentials
```

---

## Flow C: API Consumer → API Gateway (OAuth 2.0)

### Overview

Internal applications call APIs through the Gateway using OAuth 2.0 Bearer tokens.

```
Corporate Zone          Inner DMZ           Core Zone
┌─────────────┐      ┌───────────┐      ┌──────────────┐
│  API        │──C1─►│  OpenAM   │      │              │
│  Consumer   │◄─────│   SSO     │◄─C3──│  API Gateway │
│  App        │              │          │              │
└─────────────┘              │          └──────────────┘
       │                     │                  │
       └─────────C2──────────┼─────────────────►│
              (Bearer Token) │                  │
                             │                  ▼
                             │          ┌──────────────┐
                             │          │  Backend API │
                             │          └──────────────┘
```

### Flow Steps

| Step | Action | Details |
|------|--------|---------|
| **C1** | Get Token | App requests access token from OpenAM (client_credentials grant) |
| **C2** | Call API | App sends request to Gateway with Bearer token |
| **C3** | Validate Token | Gateway validates token via introspection or JWKS |
| **C4** | Rate Limit | Gateway checks rate limits in Redis |
| **C5** | Route | Gateway routes request to backend service |
| **C6** | Response | Backend returns response through Gateway to App |

### OAuth 2.0 Token Request

```http
POST /oauth2/token HTTP/1.1
Host: sso.inner-dmz.company.com
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=my-internal-app
&client_secret=xxxxxxxxxxxxx
&scope=api:read api:write
```

### Token Response

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "api:read api:write"
}
```

### API Request with Token

```http
POST /api/v1/users HTTP/1.1
Host: gravitee-gateway:8082
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
X-Correlation-ID: abc-123

{
  "name": "John Doe",
  "email": "john@company.com"
}
```

### Mermaid Diagram

```mermaid
sequenceDiagram
    autonumber
    participant App as 📱 Consumer App
    participant OpenAM as 🔐 OpenAM
    participant GW as ⚡ API Gateway
    participant Backend as 🔧 Backend
    
    App->>OpenAM: POST /oauth2/token<br/>client_credentials
    OpenAM-->>App: {access_token: "eyJ..."}
    
    App->>GW: POST /api/users<br/>Authorization: Bearer eyJ...
    GW->>OpenAM: Validate token (introspect/JWKS)
    OpenAM-->>GW: Token valid ✅
    GW->>Backend: Route request
    Backend-->>GW: 201 Created
    GW-->>App: 201 Created
```

---

## OpenAM OAuth 2.0 Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/oauth2/authorize` | GET | Authorization endpoint (user login redirect) |
| `/oauth2/token` | POST | Token endpoint (exchange code or get token) |
| `/oauth2/introspect` | POST | Token introspection (validate token) |
| `/oauth2/jwks` | GET | JSON Web Key Set (for JWT signature validation) |
| `/oauth2/userinfo` | GET | Get user information from token |
| `/oauth2/revoke` | POST | Revoke token |

---

## JWT Token Structure

```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT",
    "kid": "key-id-123"
  },
  "payload": {
    "sub": "my-internal-app",
    "iss": "https://sso.inner-dmz.company.com/openam",
    "aud": "gravitee-gateway",
    "exp": 1739462400,
    "iat": 1739458800,
    "scope": "api:read api:write",
    "client_id": "my-internal-app",
    "roles": ["SERVICE", "API_CONSUMER"]
  }
}
```

---

## Gateway JWT Validation Policy

```yaml
# Gravitee Gateway - JWT Policy Configuration
jwt:
  enabled: true
  publicKeyResolver: JWKS_URL
  resolverParameter: https://sso.inner-dmz.company.com/oauth2/jwks
  useSystemProxy: false
  extractClaims: true
  propagateAuthHeader: true
  
  # Claims to extract
  claims:
    - sub
    - scope
    - client_id
    - roles
```

---

## Complete Flow Diagram

```mermaid
flowchart TB
    subgraph CorporateZone["CORPORATE ZONE"]
        CorpUser["👤 Corporate User"]
        AppDev["👨‍💻 App Developer"]
        ConsumerApp["📱 API Consumer"]
    end

    subgraph InnerDMZ["INNER DMZ"]
        OpenAM["🔐 OpenAM SSO"]
    end

    subgraph CoreZone["CORE ZONE"]
        Console["💻 Console UI"]
        Portal["📚 Developer Portal"]
        Gateway["⚡ API Gateway"]
        Backend["🔧 Backend APIs"]
    end

    CorpUser -->|"A: Login"| Console
    Console -->|"Auth"| OpenAM
    
    AppDev -->|"B: Login"| Portal
    Portal -->|"Auth"| OpenAM
    
    ConsumerApp -->|"C1: Get Token"| OpenAM
    ConsumerApp -->|"C2: API Call"| Gateway
    Gateway -->|"C3: Validate"| OpenAM
    Gateway -->|"C4: Route"| Backend
```

---

## Related Documents

| Document | Description |
|----------|-------------|
| [Internal Gateway Architecture](./INTERNAL-GATEWAY-ARCHITECTURE.md) | Gateway infrastructure |
| [Firewall Rules](./FIREWALL_RULES_MULTI_REGION.md) | Network security |

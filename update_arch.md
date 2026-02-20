# Core Zone - HLD Architecture (Mermaid Diagrams)

## 1. Architecture Overview

```mermaid
flowchart TB
    %% ==================== STYLING ====================
    classDef internalSys fill:#455a64,stroke:#263238,color:#fff,stroke-width:2px
    classDef globalLB fill:#1a237e,stroke:#0d47a1,color:#fff,stroke-width:2px
    classDef regionalLB fill:#00695c,stroke:#004d40,color:#fff,stroke-width:2px
    classDef k8sCluster fill:#0277bd,stroke:#01579b,color:#fff,stroke-width:2px
    classDef gateway fill:#1565c0,stroke:#0d47a1,color:#fff,stroke-width:2px
    classDef controlPlane fill:#6a1b9a,stroke:#4a148c,color:#fff,stroke-width:2px
    classDef mgmtComp fill:#9c27b0,stroke:#7b1fa2,color:#fff,stroke-width:2px
    classDef redis fill:#c62828,stroke:#b71c1c,color:#fff,stroke-width:2px
    classDef elasticsearch fill:#f57f17,stroke:#e65100,color:#000,stroke-width:2px
    classDef mysql fill:#4527a0,stroke:#311b92,color:#fff,stroke-width:2px
    classDef identity fill:#d84315,stroke:#bf360c,color:#fff,stroke-width:2px
    classDef aac fill:#00838f,stroke:#006064,color:#fff,stroke-width:2px
    classDef backend fill:#37474f,stroke:#263238,color:#fff,stroke-width:2px

    %% ==================== DATA PLANE ====================
    subgraph DataPlane["DATA PLANE - Internal API Gateway"]
        direction TB
        
        InternalSys["Internal Systems<br/>(Microservices, Apps)"]:::internalSys
        
        GLB["Global Load Balancer<br/>(GION LS / GLB)<br/>Geo-routing | Health Checks"]:::globalLB
        
        subgraph US["US REGION"]
            direction TB
            US_RLB["Regional LB<br/>(US)"]:::regionalLB
            subgraph US_K8S["Kubernetes Cluster (US)"]
                US_Node1["Worker Node 1<br/>Gravitee Gateway<br/>Replicas: 2"]:::gateway
                US_Node2["Worker Node 2<br/>Gravitee Gateway<br/>Replicas: 2"]:::gateway
            end
        end
        
        subgraph EU["EU REGION"]
            direction TB
            EU_RLB["Regional LB<br/>(EU)"]:::regionalLB
            subgraph EU_K8S["Kubernetes Cluster (EU)"]
                EU_Node1["Worker Node 1<br/>Gravitee Gateway<br/>Replicas: 2"]:::gateway
                EU_Node2["Worker Node 2<br/>Gravitee Gateway<br/>Replicas: 2"]:::gateway
            end
        end
        
        subgraph APAC["APAC REGION"]
            direction TB
            APAC_RLB["Regional LB<br/>(APAC)"]:::regionalLB
            subgraph APAC_K8S["Kubernetes Cluster (APAC)"]
                APAC_Node1["Worker Node 1<br/>Gravitee Gateway<br/>Replicas: 2"]:::gateway
                APAC_Node2["Worker Node 2<br/>Gravitee Gateway<br/>Replicas: 2"]:::gateway
            end
        end
        
        AAC["Gravitee AAC<br/>Access & Authorization<br/>Token Validation | OAuth2"]:::aac
        
        Backend["Internal API Backends<br/>Microservices | Legacy | Databases"]:::backend
    end

    %% ==================== CONTROL PLANE ====================
    subgraph ControlPlane["CONTROL PLANE (Shared)"]
        direction TB
        
        subgraph MgmtK8s["Kubernetes Cluster - Management"]
            MgmtUI["Management UI<br/>(Console)"]:::mgmtComp
            MgmtAPI["Management API<br/>(REST)"]:::mgmtComp
            DevPortal["Developer Portal<br/>(Catalog)"]:::mgmtComp
        end
        
        subgraph DataServices["Shared Data Services"]
            MySQL["MySQL Database<br/>API Definitions<br/>Plans & Subscriptions"]:::mysql
            Redis["Redis (Shared)<br/>Rate Limiting<br/>Distributed Cache"]:::redis
            ES["Elasticsearch<br/>Analytics & Metrics<br/>Access Logs"]:::elasticsearch
        end
    end

    %% ==================== IDENTITY LAYER ====================
    subgraph IdentityLayer["IDENTITY LAYER"]
        direction LR
        SSO["SSO Provider<br/>(SAML/OIDC)"]:::identity
        LDAP["LDAP/AD<br/>(User Dir)"]:::identity
        Keycloak["Keycloak<br/>(IAM)"]:::identity
    end

    %% ==================== TRAFFIC FLOW CONNECTIONS ====================
    InternalSys -->|"1. API Request"| GLB
    GLB -->|"2. Geo-routing"| US_RLB
    GLB -->|"2. Geo-routing"| EU_RLB
    GLB -->|"2. Geo-routing"| APAC_RLB
    
    US_RLB -->|"3"| US_Node1
    US_RLB -->|"3"| US_Node2
    EU_RLB -->|"3"| EU_Node1
    EU_RLB -->|"3"| EU_Node2
    APAC_RLB -->|"3"| APAC_Node1
    APAC_RLB -->|"3"| APAC_Node2
    
    US_Node1 -.->|"4. Token"| AAC
    US_Node1 -.->|"5. Rate/Cache"| Redis
    US_Node1 -->|"6. API Call"| Backend
    US_Node1 -.->|"7. Analytics"| ES
    
    MgmtAPI -.->|"Config Sync"| US_Node1
    MgmtUI --> MgmtAPI
    MgmtAPI --> MySQL
    MgmtAPI -.-> SSO
```

---

## 2. API Traffic Flow (Sequence)

```mermaid
sequenceDiagram
    autonumber
    
    participant Client as Internal Systems
    participant GLB as Global LB
    participant RLB as Regional LB
    participant GW as Gateway Pod
    participant AAC as AAC
    participant Redis as Redis
    participant Backend as Backend API
    participant ES as Elasticsearch
    
    Note over Client,ES: API REQUEST FLOW
    
    rect rgb(230, 245, 255)
        Client->>GLB: 1. HTTPS Request<br/>POST /api/v1/users
    end
    
    rect rgb(255, 245, 230)
        GLB->>RLB: 2. Geo-route to nearest region
    end
    
    rect rgb(230, 255, 230)
        RLB->>GW: 3. Load balance to Gateway Pod
    end
    
    rect rgb(255, 230, 245)
        GW->>AAC: 4. Validate JWT/OAuth2 token
        AAC-->>GW: Token Valid + Claims
    end
    
    rect rgb(245, 230, 230)
        GW->>Redis: 5. Check rate limit & cache
        Redis-->>GW: Rate OK / Cache MISS
    end
    
    rect rgb(230, 255, 255)
        GW->>Backend: 6. Forward request
        Backend-->>GW: Response (200 OK)
    end
    
    rect rgb(245, 245, 230)
        GW--)ES: 7. Async analytics
    end
    
    GW-->>Client: Return Response
```

---

## 3. Management Login Flow

```mermaid
sequenceDiagram
    autonumber
    
    participant User as Admin User
    participant UI as Management UI
    participant API as Management API
    participant SSO as SSO/LDAP
    participant DB as Database
    
    Note over User,DB: MANAGEMENT LOGIN FLOW
    
    rect rgb(230, 245, 255)
        User->>UI: A. Navigate to Console
        UI-->>User: Display Login Page
    end
    
    rect rgb(255, 245, 230)
        User->>UI: B. Submit credentials
        UI->>API: C. POST /auth/login
    end
    
    rect rgb(230, 255, 230)
        API->>SSO: D. Validate credentials
        SSO-->>API: E. User info + roles
    end
    
    rect rgb(255, 230, 245)
        API->>DB: F. Store session
        API-->>UI: G. Return JWT token
    end
    
    rect rgb(245, 245, 230)
        UI-->>User: H. Set cookie & redirect
    end
```

---

## 4. Data Plane - Regional Architecture

```mermaid
flowchart LR
    subgraph GLB_Layer["Global Load Balancer Layer"]
        GLB["GION LS / GLB<br/>━━━━━━━━━━━━<br/>• Geo-DNS routing<br/>• Health monitoring<br/>• Automatic failover<br/>• SSL termination"]
    end
    
    subgraph US_Region["US Region"]
        direction TB
        US_RLB["Regional LB (US)<br/>L4/L7 | Sticky Sessions"]
        subgraph US_K8s["K8s Cluster"]
            US_W1["Node 1"]
            US_W2["Node 2"]
            US_GW1["GW Pod 1-2"]
            US_GW2["GW Pod 3-4"]
        end
        US_RLB --> US_W1 --> US_GW1
        US_RLB --> US_W2 --> US_GW2
    end
    
    subgraph EU_Region["EU Region"]
        direction TB
        EU_RLB["Regional LB (EU)<br/>L4/L7 | Sticky Sessions"]
        subgraph EU_K8s["K8s Cluster"]
            EU_W1["Node 1"]
            EU_W2["Node 2"]
            EU_GW1["GW Pod 1-2"]
            EU_GW2["GW Pod 3-4"]
        end
        EU_RLB --> EU_W1 --> EU_GW1
        EU_RLB --> EU_W2 --> EU_GW2
    end
    
    subgraph APAC_Region["APAC Region"]
        direction TB
        APAC_RLB["Regional LB (APAC)<br/>L4/L7 | Sticky Sessions"]
        subgraph APAC_K8s["K8s Cluster"]
            APAC_W1["Node 1"]
            APAC_W2["Node 2"]
            APAC_GW1["GW Pod 1-2"]
            APAC_GW2["GW Pod 3-4"]
        end
        APAC_RLB --> APAC_W1 --> APAC_GW1
        APAC_RLB --> APAC_W2 --> APAC_GW2
    end
    
    GLB --> US_RLB
    GLB --> EU_RLB
    GLB --> APAC_RLB
    
    style GLB fill:#1a237e,color:#fff
    style US_RLB fill:#00695c,color:#fff
    style EU_RLB fill:#00695c,color:#fff
    style APAC_RLB fill:#00695c,color:#fff
```

---

## 5. Control Plane Architecture

```mermaid
flowchart TB
    subgraph ControlPlane["CONTROL PLANE KUBERNETES CLUSTER<br/>(Shared - All Regions)"]
        direction TB
        
        subgraph Management["Management Components"]
            direction LR
            MUI["Management UI<br/>(Console)"]
            MAPI["Management API<br/>(REST API)"]
            Portal["Developer Portal<br/>(API Catalog)"]
        end
        
        subgraph DataLayer["SHARED DATA SERVICES"]
            direction LR
            
            subgraph MySQL_Box["MySQL"]
                MySQL["MySQL Database<br/>━━━━━━━━━━━━<br/>• API Definitions<br/>• Plans<br/>• Subscriptions"]
            end
            
            subgraph Redis_Box["Redis"]
                Redis["Redis (Shared)<br/>━━━━━━━━━━━━<br/>• Rate Limiting<br/>• Caching<br/>• Sessions"]
            end
            
            subgraph ES_Box["Elasticsearch"]
                ES["Elasticsearch<br/>━━━━━━━━━━━━<br/>• API Metrics<br/>• Access Logs<br/>• Audit Trail"]
            end
        end
        
        MUI --> MAPI
        MAPI --> MySQL
        MAPI --> Redis
        MAPI --> ES
    end
    
    subgraph Identity["IDENTITY LAYER"]
        direction LR
        SSO["SSO Provider<br/>SAML / OIDC"]
        LDAP["LDAP/AD<br/>User Directory"]
        KC["Keycloak<br/>IAM"]
    end
    
    MAPI -.-> SSO
    MAPI -.-> LDAP
    MAPI -.-> KC
    
    style MySQL fill:#4527a0,color:#fff
    style Redis fill:#c62828,color:#fff
    style ES fill:#f57f17,color:#000
    style SSO fill:#d84315,color:#fff
    style LDAP fill:#bf360c,color:#fff
    style KC fill:#e65100,color:#fff
```

---

## 6. Security Layers

```mermaid
flowchart TB
    subgraph SecurityLayers["SECURITY ARCHITECTURE"]
        direction TB
        
        L1["Layer 1: NETWORK<br/>━━━━━━━━━━━━━━━━<br/>TLS 1.3 | mTLS | Network Policies"]
        L2["Layer 2: AUTHENTICATION<br/>━━━━━━━━━━━━━━━━<br/>SSO | LDAP | OAuth2 | JWT"]
        L3["Layer 3: AUTHORIZATION<br/>━━━━━━━━━━━━━━━━<br/>RBAC | Policies | Scopes"]
        L4["Layer 4: API SECURITY<br/>━━━━━━━━━━━━━━━━<br/>Rate Limiting | Quotas"]
        L5["Layer 5: DATA SECURITY<br/>━━━━━━━━━━━━━━━━<br/>Encryption | Masking"]
        
        L1 --> L2 --> L3 --> L4 --> L5
    end
    
    style L1 fill:#1565c0,color:#fff
    style L2 fill:#2e7d32,color:#fff
    style L3 fill:#7b1fa2,color:#fff
    style L4 fill:#c62828,color:#fff
    style L5 fill:#00695c,color:#fff
```

---

## 7. Configuration Sync Flow

```mermaid
flowchart TB
    subgraph ControlPlane["Control Plane"]
        MAPI["Management API"]
        MySQL["MySQL<br/>(Config Store)"]
        MAPI --> MySQL
    end
    
    MAPI -.->|"Config Push/Pull<br/>(Every 5 seconds)"| US_GW
    MAPI -.->|"Config Push/Pull"| EU_GW
    MAPI -.->|"Config Push/Pull"| APAC_GW
    
    subgraph DataPlane["Data Plane Gateways"]
        direction LR
        US_GW["US Gateways<br/>(4 pods)"]
        EU_GW["EU Gateways<br/>(4 pods)"]
        APAC_GW["APAC Gateways<br/>(4 pods)"]
    end
    
    style MAPI fill:#7b1fa2,color:#fff
    style MySQL fill:#4527a0,color:#fff
    style US_GW fill:#1565c0,color:#fff
    style EU_GW fill:#1565c0,color:#fff
    style APAC_GW fill:#1565c0,color:#fff
```

---

## Legend

| Symbol | Meaning |
|--------|---------|
| `-->` | Synchronous request/response |
| `-.->` | Async/optional connection |
| `--)` | Fire-and-forget (async) |
| Numbered circles (1-7) | API Traffic Flow steps |
| Lettered circles (A-H) | Login Flow steps |

---

## Flow Summary

### API Traffic Flow Steps
| Step | From | To | Action |
|------|------|----|--------|
| **1** | Internal Systems | Global LB | Initial request |
| **2** | Global LB | Regional LB | Geo-routing |
| **3** | Regional LB | K8s Gateway | Load balancing |
| **4** | Gateway | AAC | Token validation |
| **5** | Gateway | Redis | Rate limit + cache |
| **6** | Gateway | Backend | API invocation |
| **7** | Gateway | Elasticsearch | Analytics (async) |

### Login Flow Steps
| Step | From | To | Action |
|------|------|----|--------|
| **A** | User | UI | Navigate to console |
| **B** | User | UI | Submit credentials |
| **C** | UI | API | Forward auth request |
| **D** | API | SSO/LDAP | Validate credentials |
| **E** | SSO/LDAP | API | Return user info |
| **F** | API | Database | Store session |
| **G** | API | UI | Return JWT token |
| **H** | UI | User | Set cookie & redirect |

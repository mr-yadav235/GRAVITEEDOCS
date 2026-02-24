# Internal API Gateway Architecture

## Overview

This document details the Internal API Gateway architecture for handling internal microservice-to-microservice communication and internal corporate user access. **All components reside within the Core Zone** - there is no DMZ layer for internal gateway deployments.

![Internal Gateway Architecture](./diagrams/HLD-CORE-ZONE-ARCHITECTURE.svg)

---

## 1. Architecture Principles

### 1.1 Core Zone Only Deployment

For Internal API Gateway, **everything is deployed within the Core Zone**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CORE ZONE                                       │
│                    (Single Zone - No DMZ Required)                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   USERS & CONSUMERS                                                          │
│   ├── Corporate Users (API Developers, Platform Team, App Developers)       │
│   └── Internal API Consumer Applications (Microservices)                    │
│                                                                              │
│   IDENTITY & ACCESS                                                          │
│   └── OpenAM SSO Server (OAuth 2.0 / OIDC)                                  │
│                                                                              │
│   GRAVITEE CONTROL PLANE (Kubernetes Cluster)                               │
│   ├── Console UI (Management Interface)                                      │
│   ├── Management API (REST API)                                             │
│   ├── Developer Portal (API Catalog)                                        │
│   ├── Redis Cluster (Rate Limiting, Caching) ← IN-CLUSTER                   │
│   └── Elasticsearch Cluster (Analytics, Logs) ← IN-CLUSTER                  │
│                                                                              │
│   GRAVITEE DATA PLANE                                                        │
│   ├── API Gateway - US Region (2 Nodes)                                     │
│   ├── API Gateway - EU Region (2 Nodes)                                     │
│   └── API Gateway - ASIA Region (2 Nodes)                                   │
│                                                                              │
│   DATA SERVICES (External to K8s Cluster)                                   │
│   └── MongoDB Cluster (API Configs, Plans, Subscriptions) ← EXTERNAL       │
│                                                                              │
│   BACKEND SERVICES                                                           │
│   └── Internal Microservices / Backend APIs                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Why Core Zone Only?

| Reason | Explanation |
|--------|-------------|
| Internal Traffic Only | No external/public API exposure required |
| No Firewall Required | All components communicate within same zone |
| Simplified Architecture | No DMZ complexity, reduced network hops |
| Lower Latency | All components in same zone |
| Easier Management | Single zone to manage and monitor |
| Cost Effective | No duplicate infrastructure across zones |

### 1.3 Data Services Placement

| Service | Location | Reason |
|---------|----------|--------|
| MongoDB | External to K8s (Core Zone) | Dedicated resources, independent scaling, existing DBA management |
| Redis | Inside Control Plane K8s | Low-latency access for rate limiting, co-located with gateway |
| Elasticsearch | Inside Control Plane K8s | Analytics and logging, managed alongside application |

---

## 2. Component Layout

### 2.1 Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                     CORE ZONE                                            │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  ┌───────────────────────────────────────────────────────────────────────────────────┐  │
│  │                           CORPORATE USERS                                          │  │
│  │  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────────────┐   │  │
│  │  │ API         │   │ Platform    │   │ App         │   │ API Consumer        │   │  │
│  │  │ Developers  │   │ Team        │   │ Developers  │   │ Applications        │   │  │
│  │  │ (Console)   │   │ (Admin)     │   │ (Portal)    │   │ (Microservices)     │   │  │
│  │  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └──────────┬──────────┘   │  │
│  │         │                 │                 │                      │              │  │
│  └─────────┼─────────────────┼─────────────────┼──────────────────────┼──────────────┘  │
│            │                 │                 │                      │                 │
│            ▼                 ▼                 ▼                      │                 │
│  ┌───────────────────────────────────────────────────────────────┐   │                 │
│  │                    OPENAM SSO SERVER                          │   │                 │
│  │           OAuth 2.0 / OIDC Provider (Core Zone)               │   │                 │
│  │  /oauth2/authorize | /oauth2/token | /oauth2/introspect       │   │                 │
│  │  Connected to Corporate LDAP/Active Directory                 │   │                 │
│  └───────────────────────────────────────────────────────────────┘   │                 │
│            │                 │                 │                      │                 │
│            ▼                 ▼                 ▼                      │                 │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │                 GRAVITEE CONTROL PLANE (Kubernetes Cluster)                      │  │
│  │  ┌─────────────┐   ┌─────────────────┐   ┌─────────────────┐                    │  │
│  │  │ Console UI  │   │ Management API  │   │ Developer Portal│                    │  │
│  │  │ :443        │   │ :8083           │   │ :443            │                    │  │
│  │  └─────────────┘   └────────┬────────┘   └─────────────────┘                    │  │
│  │                             │                                                    │  │
│  │  ┌──────────────────────────┴──────────────────────────┐                        │  │
│  │  │           DATA SERVICES (IN-CLUSTER)                 │                        │  │
│  │  │  ┌─────────────────┐       ┌─────────────────────┐  │                        │  │
│  │  │  │ Redis Cluster   │       │ Elasticsearch       │  │                        │  │
│  │  │  │ :6379 (6 nodes) │       │ :9200 (3 nodes)     │  │                        │  │
│  │  │  │ Rate Limiting   │       │ Analytics/Logs      │  │                        │  │
│  │  │  └─────────────────┘       └─────────────────────┘  │                        │  │
│  │  └─────────────────────────────────────────────────────┘                        │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                        │                                               │
│                                        │ Config Read/Write                             │
│                                        ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │               MONGODB CLUSTER (External to K8s - Core Zone)                      │  │
│  │  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐                            │  │
│  │  │ Primary     │   │ Secondary 1 │   │ Secondary 2 │   API Configs, Plans,      │  │
│  │  │ :27017      │   │ :27017      │   │ :27017      │   Subscriptions, Users     │  │
│  │  └─────────────┘   └─────────────┘   └─────────────┘                            │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                        │                                               │
│                                        │ Config Sync (Pull)                            │
│                                        ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │                          GRAVITEE DATA PLANE                                     │  │
│  │                                                                                  │  │
│  │        ┌────────────────┐                                                        │  │
│  │        │  Global LB     │◄─────────────────────────────────────────────┐        │  │
│  │        │  (Internal)    │                                              │        │  │
│  │        └───────┬────────┘                                              │        │  │
│  │                │                                                       │        │  │
│  │    ┌───────────┼───────────┬───────────────────┐                      │        │  │
│  │    ▼           ▼           ▼                   │                      │        │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐       │                      │        │  │
│  │  │US Region │ │EU Region │ │ASIA Reg. │       │                      │        │  │
│  │  │GW (2)    │ │GW (2)    │ │GW (2)    │       │        API           │        │  │
│  │  │:8082     │ │:8082     │ │:8082     │       │      Consumers       │        │  │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘       │                      │        │  │
│  │       │            │            │              │                      │        │  │
│  │       └────────────┼────────────┘              │                      │        │  │
│  │                    │                           │                      │        │  │
│  └────────────────────┼───────────────────────────┼──────────────────────┘        │  │
│                       ▼                           │                               │  │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │                         BACKEND SERVICES                                         │  │
│  │           Internal Microservices | Legacy Systems | Databases                   │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Details

### 3.1 Corporate Users (Core Zone)

| User Type | Access | Purpose |
|-----------|--------|---------|
| API Developers | Console UI | Create and manage APIs |
| Platform Team | Console UI (Admin) | Platform administration, RBAC |
| App Developers | Developer Portal | Discover APIs, manage subscriptions |
| API Consumer Apps | API Gateway | Invoke internal APIs |

### 3.2 OpenAM SSO Server (Core Zone)

| Property | Value |
|----------|-------|
| Location | Core Zone |
| Protocol | HTTPS (:443) |
| Function | OAuth 2.0 / OIDC Provider |
| Integration | Corporate LDAP/Active Directory |

**OAuth Endpoints:**
- `/oauth2/authorize` - Authorization endpoint
- `/oauth2/token` - Token endpoint
- `/oauth2/introspect` - Token introspection
- `/oauth2/userinfo` - User information
- `/oauth2/connect/jwk_uri` - JWKS endpoint

### 3.3 Gravitee Control Plane (Kubernetes Cluster - Core Zone)

| Component | Port | Location | Purpose |
|-----------|------|----------|---------|
| Console UI | 443 | In-Cluster | Web-based management interface |
| Management API | 8083 | In-Cluster | REST API for configuration |
| Developer Portal | 443 | In-Cluster | API discovery and subscription |
| Redis Cluster | 6379 | In-Cluster | Rate limiting, distributed caching |
| Elasticsearch | 9200 | In-Cluster | Analytics, logs, metrics |

### 3.4 MongoDB Cluster (External to K8s - Core Zone)

| Property | Value |
|----------|-------|
| Location | Core Zone (External to K8s) |
| Port | 27017 |
| Topology | 3-node Replica Set |
| Purpose | API configurations, plans, subscriptions, users |

**Why External to K8s?**
- Dedicated resources for database workloads
- Independent scaling from application tier
- Managed by existing DBA team
- Separate backup and recovery procedures
- Better resource isolation

### 3.5 Gravitee Data Plane (Core Zone)

| Region | Nodes | Port | Purpose |
|--------|-------|------|---------|
| US | 2 | 8082 | US region internal traffic |
| EU | 2 | 8082 | EU region internal traffic |
| ASIA | 2 | 8082 | ASIA region internal traffic |
| **Total** | **6** | - | All internal API traffic |

---

## 4. Communication Flows

### 4.1 User Authentication Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    USER AUTHENTICATION (ALL IN CORE ZONE)                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────┐                                                          │
│   │ Corporate    │                                                          │
│   │ User         │                                                          │
│   │ (Core Zone)  │                                                          │
│   └──────┬───────┘                                                          │
│          │                                                                   │
│          │ 1. Access Console/Portal                                         │
│          ▼                                                                   │
│   ┌──────────────┐         2. Redirect to OAuth         ┌──────────────┐   │
│   │ Console UI / │ ─────────────────────────────────────►│ OpenAM SSO   │   │
│   │ Portal       │                                       │ (Core Zone)  │   │
│   │ (Core Zone)  │◄─────────────────────────────────────│              │   │
│   └──────────────┘   4. Return with Auth Code + Token   └──────┬───────┘   │
│          │                                                      │           │
│          │                                               3. User Login      │
│          │                                               (LDAP/AD Auth)     │
│          │                                                      │           │
│          │ 5. Access with JWT                                   ▼           │
│          ▼                                               ┌──────────────┐   │
│   ┌──────────────┐                                       │ LDAP / AD    │   │
│   │ Management   │                                       │ (Core Zone)  │   │
│   │ API          │                                       └──────────────┘   │
│   │ (Core Zone)  │                                                          │
│   └──────────────┘                                                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 API Request Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    API REQUEST FLOW (ALL IN CORE ZONE)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────┐                                                          │
│   │ API Consumer │                                                          │
│   │ Application  │                                                          │
│   │ (Core Zone)  │                                                          │
│   └──────┬───────┘                                                          │
│          │                                                                   │
│          │ 1. Get OAuth Token (Client Credentials)                          │
│          ▼                                                                   │
│   ┌──────────────┐                                                          │
│   │ OpenAM SSO   │────────────────────────────────────────┐                 │
│   │ (Core Zone)  │                                        │                 │
│   └──────┬───────┘                                        │                 │
│          │                                                │                 │
│          │ 2. Return Access Token                         │                 │
│          ▼                                                │                 │
│   ┌──────────────┐                                        │                 │
│   │ API Consumer │                                        │                 │
│   │ Application  │                                        │                 │
│   └──────┬───────┘                                        │                 │
│          │                                                │                 │
│          │ 3. API Request with Bearer Token               │                 │
│          ▼                                                │                 │
│   ┌──────────────┐         4. Validate Token              │                 │
│   │ API Gateway  │────────────────────────────────────────┘                 │
│   │ (Core Zone)  │         (JWKS / Introspect)                              │
│   └──────┬───────┘                                                          │
│          │                                                                   │
│          │ 5. Forward Request (if valid)                                    │
│          ▼                                                                   │
│   ┌──────────────┐                                                          │
│   │ Backend API  │                                                          │
│   │ (Core Zone)  │                                                          │
│   └──────┬───────┘                                                          │
│          │                                                                   │
│          │ 6. Response                                                      │
│          ▼                                                                   │
│   ┌──────────────┐                                                          │
│   │ API Consumer │                                                          │
│   └──────────────┘                                                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 Configuration Sync Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONFIG SYNC FLOW (ALL IN CORE ZONE)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────┐         1. API Published          ┌──────────────┐       │
│   │ Admin User   │ ──────────────────────────────────►│ Console UI   │       │
│   │ (Core Zone)  │                                   │ (Core Zone)  │       │
│   └──────────────┘                                   └──────┬───────┘       │
│                                                             │               │
│                                              2. Save to Management API      │
│                                                             ▼               │
│                                                      ┌──────────────┐       │
│                                                      │ Management   │       │
│                                                      │ API          │       │
│                                                      └──────┬───────┘       │
│                                                             │               │
│                                              3. Store Config                │
│                                                             ▼               │
│                                                      ┌──────────────┐       │
│                                                      │   MongoDB    │       │
│                                                      │ (External)   │       │
│                                                      │  Core Zone   │       │
│                                                      └──────┬───────┘       │
│                                                             │               │
│   ┌──────────────┐         4. Pull Config (Sync)           │               │
│   │ API Gateway  │◄─────────────────────────────────────────┘               │
│   │ US Region    │                                                          │
│   └──────────────┘                                                          │
│                                                                              │
│   ┌──────────────┐         4. Pull Config (Sync)                            │
│   │ API Gateway  │◄─────────────────────────────────────────┘               │
│   │ EU Region    │                                                          │
│   └──────────────┘                                                          │
│                                                                              │
│   ┌──────────────┐         4. Pull Config (Sync)                            │
│   │ API Gateway  │◄─────────────────────────────────────────┘               │
│   │ ASIA Region  │                                                          │
│   └──────────────┘                                                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Network Configuration

### 5.1 Internal Network (Core Zone Only)

Since everything is in the Core Zone, network configuration is simplified. **No firewall rules required** between components.

| Network Segment | CIDR Example | Components |
|-----------------|--------------|------------|
| Core Zone | 10.100.0.0/16 | All Gravitee components |
| User Subnet | 10.100.1.0/24 | Corporate users, workstations |
| Identity Subnet | 10.100.2.0/24 | OpenAM, LDAP/AD |
| Control Plane Subnet | 10.100.10.0/24 | Console, Management API, Portal, Redis, ES |
| Data Plane Subnet | 10.100.20.0/24 | API Gateway nodes |
| MongoDB Subnet | 10.100.30.0/24 | MongoDB cluster (external to K8s) |
| Backend Subnet | 10.100.100.0/24 | Backend microservices |

### 5.2 Port Reference

All communication is internal within the Core Zone:

| Source | Destination | Port | Protocol | Purpose |
|--------|-------------|------|----------|---------|
| Users | Console UI | 443 | HTTPS | Management access |
| Users | Portal | 443 | HTTPS | API discovery |
| Users | OpenAM | 443 | HTTPS | Authentication |
| API Consumers | Gateway | 8082 | HTTPS | API calls |
| Gateway | OpenAM | 443 | HTTPS | Token validation |
| Gateway | MongoDB | 27017 | TLS | Config sync |
| Gateway | Redis | 6379 | TLS | Rate limiting |
| Gateway | Elasticsearch | 9200 | HTTPS | Analytics |
| Gateway | Backend APIs | Various | HTTPS/mTLS | API forwarding |
| Management API | MongoDB | 27017 | TLS | Configuration |
| Management API | Elasticsearch | 9200 | HTTPS | Analytics |
| Management API | OpenAM | 443 | HTTPS | User auth |

### 5.3 No Firewall Required

```
┌─────────────────────────────────────────────────────────────────┐
│                    SIMPLIFIED ARCHITECTURE                       │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                          │   │
│   │                      CORE ZONE                           │   │
│   │              (No Firewall Between Components)            │   │
│   │                                                          │   │
│   │    ┌──────────┐  ┌──────────┐  ┌──────────┐            │   │
│   │    │ Users    │  │ OpenAM   │  │ Gravitee │            │   │
│   │    └──────────┘  └──────────┘  │ Control  │            │   │
│   │                                 │ Plane    │            │   │
│   │                                 │(+Redis/ES)│           │   │
│   │                                 └──────────┘            │   │
│   │                                                          │   │
│   │    ┌──────────┐  ┌──────────┐  ┌──────────┐            │   │
│   │    │ Backends │  │ MongoDB  │  │ Gateway  │            │   │
│   │    │          │  │(External)│  │ Clusters │            │   │
│   │    └──────────┘  └──────────┘  └──────────┘            │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   ✓ NO OUTER DMZ    ✓ NO INNER DMZ    ✓ NO FIREWALL RULES       │
│   ✓ SINGLE ZONE DEPLOYMENT    ✓ ALL INTERNAL TRAFFIC            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Security Configuration

### 6.1 TLS Configuration

All internal communication uses TLS:

```yaml
# Gateway TLS Configuration
http:
  ssl:
    enabled: true
    keystore:
      type: PKCS12
      path: /certs/gateway-keystore.p12
      password: ${KEYSTORE_PASSWORD}
    truststore:
      type: PKCS12
      path: /certs/truststore.p12
      password: ${TRUSTSTORE_PASSWORD}
    protocols:
      - TLSv1.2
      - TLSv1.3
```

### 6.2 OAuth Configuration (OpenAM in Core Zone)

```yaml
# Management API - OpenAM Configuration
security:
  providers:
    - type: oidc
      id: openam
      clientId: gravitee-console
      clientSecret: ${OPENAM_CLIENT_SECRET}
      # OpenAM is in Core Zone - internal URL
      tokenEndpoint: https://openam.core.internal/oauth2/access_token
      authorizeEndpoint: https://openam.core.internal/oauth2/authorize
      userInfoEndpoint: https://openam.core.internal/oauth2/userinfo
      jwksUri: https://openam.core.internal/oauth2/connect/jwk_uri
```

### 6.3 mTLS for Service-to-Service

```yaml
# mTLS for internal services
ssl:
  clientAuth: required
  keystore:
    path: /certs/service-keystore.p12
  truststore:
    path: /certs/internal-ca-truststore.p12
```

### 6.4 MongoDB Connection (External)

```yaml
# MongoDB connection from Control Plane
management:
  mongodb:
    uri: mongodb://mongodb-primary.core.internal:27017,mongodb-secondary1.core.internal:27017,mongodb-secondary2.core.internal:27017/gravitee?replicaSet=rs0
    ssl:
      enabled: true
      truststore:
        path: /certs/mongodb-truststore.p12
        password: ${MONGODB_TRUSTSTORE_PASSWORD}
```

---

## 7. High Availability

### 7.1 Component Redundancy

| Component | Replicas | Location | HA Mode |
|-----------|----------|----------|---------|
| Console UI | 2 | In-Cluster | Active-Active |
| Management API | 2 | In-Cluster | Active-Active |
| Developer Portal | 2 | In-Cluster | Active-Active |
| Redis Cluster | 6 | In-Cluster | Cluster Mode |
| Elasticsearch | 3 | In-Cluster | Cluster Mode |
| API Gateway (per region) | 2 | Core Zone | Active-Active |
| MongoDB | 3 | External (Core Zone) | Replica Set |
| OpenAM | 2 | Core Zone | Active-Active |

### 7.2 Load Balancing

```
                    ┌─────────────────┐
                    │  Internal LB    │
                    │  (Core Zone)    │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
    ┌─────────┐        ┌─────────┐        ┌─────────┐
    │ GW US-1 │        │ GW EU-1 │        │ GW AS-1 │
    │ GW US-2 │        │ GW EU-2 │        │ GW AS-2 │
    └─────────┘        └─────────┘        └─────────┘
```

---

## 8. Monitoring & Observability

### 8.1 Key Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| API Request Rate | Requests/second | > 90% capacity |
| Response Latency (P99) | 99th percentile latency | > 200ms |
| Error Rate | 4xx/5xx responses | > 1% |
| Gateway Health | Node availability | < 2 per region |
| MongoDB Replication Lag | Sync delay | > 5 seconds |
| OpenAM Token Issuance | Tokens/minute | Baseline + 50% |

### 8.2 Logging

All components log to Elasticsearch (in-cluster) for centralized analysis:

```
Component Logs → Elasticsearch (In-Cluster) → Kibana Dashboard
                          ↓
                   Alerting Rules
```

---

## 9. Summary

### Key Architecture Points

| Aspect | Decision |
|--------|----------|
| Zone Model | Single Core Zone - All components internal |
| Firewall | Not required - All traffic within Core Zone |
| MongoDB | External to K8s cluster (dedicated infrastructure) |
| Redis | Inside Control Plane K8s cluster |
| Elasticsearch | Inside Control Plane K8s cluster |
| Authentication | OpenAM SSO with OAuth 2.0/OIDC (Core Zone) |
| Gateway Deployment | Multi-region (US/EU/ASIA), 2 nodes per region |

---

## 10. Related Documents

| Document | Description |
|----------|-------------|
| [TLS & OAuth Configuration](./TLS-OAUTH-CONFIGURATION.md) | Security configuration details |
| [Deployment Guide](./DEPLOYMENT-GUIDE.md) | Kubernetes deployment steps |
| [Pre-Deployment Checklist](./PRE-DEPLOYMENT-CONNECTIVITY-CHECKLIST.md) | Connectivity verification |
| [API Group & RBAC](./API-GROUP-RBAC.md) | Access control model |
| [README](./README.md) | Documentation index |

# JSZ (Japan Security Zone) Architecture

## Overview

This document describes the architecture where the **Core Zone** (containing MongoDB and Control Plane) interacts with the **JSZ Zone** (Japan Security Zone). The JSZ Zone is divided into two sub-zones:

1. **Secure Content Provider (SCP)** - Contains the API Gateway nodes
2. **Secure Zone Data Provider (SZDP)** - Contains Redis for rate limiting

![JSZ Core Zone Architecture](./diagrams/JSZ-CORE-ZONE-ARCHITECTURE.svg)

---

## 1. Architecture Overview

### 1.1 Zone Structure

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                           │
│  ┌─────────────────────────┐  ║ F ║  ┌─────────────────────────────────────┐            │
│  │       CORE ZONE         │  ║ I ║  │           JSZ ZONE                   │            │
│  │                         │  ║ R ║  │                                      │            │
│  │  ┌───────────────────┐  │  ║ E ║  │  ┌───────────────────────────────┐  │            │
│  │  │  Control Plane    │  │  ║ W ║  │  │  Secure Content Provider (SCP) │  │            │
│  │  │  - Console UI     │  │  ║ A ║  │  │  - API Gateway Node 1          │  │            │
│  │  │  - Management API │  │  ║ L ║  │  │  - API Gateway Node 2          │  │            │
│  │  │  - Portal         │  │  ║ L ║  │  │  - Load Balancer               │  │            │
│  │  │  - OpenAM SSO     │◄─┼──╫───╫──┼─►│                                │  │            │
│  │  │  - Elasticsearch  │  │  ║   ║  │  └───────────────────────────────┘  │            │
│  │  └───────────────────┘  │  ║   ║  │                │                     │            │
│  │           ▲             │  ║   ║  │                │ Rate Limit          │            │
│  │  ┌────────┴──────────┐  │  ║   ║  │                ▼                     │            │
│  │  │  MongoDB Cluster  │◄─┼──╫───╫──┼──┐ ┌───────────────────────────────┐  │            │
│  │  │  (External to K8s)│  │  ║   ║  │  │ │ Secure Zone Data Provider     │  │            │
│  │  │  - Primary        │  │  ║   ║  │  │ │ (SZDP)                        │  │            │
│  │  │  - Secondary x2   │  │  ║   ║  │  │ │  - Redis Cluster (6 nodes)    │  │            │
│  │  └───────────────────┘  │  ║   ║  │  │ └───────────────────────────────┘  │            │
│  │                         │  ║   ║  │  │                                    │            │
│  │  ┌───────────────────┐  │  ║   ║  │  │ ┌───────────────────────────────┐  │            │
│  │  │  Backend Services │◄─┼──╫───╫──┼──┘ │  JSZ Backend Services         │  │            │
│  │  └───────────────────┘  │  ║   ║  │    └───────────────────────────────┘  │            │
│  │                         │  ║   ║  │                                      │            │
│  └─────────────────────────┘  ║   ║  └─────────────────────────────────────┘            │
│                               ║   ║                                                      │
│                               ║   ║  Firewall Rules:                                     │
│                               ║   ║  - :27017 (MongoDB)                                  │
│                               ║   ║  - :9200 (Elasticsearch)                             │
│                               ║   ║  - :443 (HTTPS/Backend APIs)                         │
│                                                                                           │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Key Architecture Decisions

| Decision | Description |
|----------|-------------|
| **MongoDB Location** | Core Zone (external to K8s) - Central configuration store |
| **Elasticsearch Location** | Core Zone (in Control Plane cluster) - Centralized analytics |
| **Redis Location** | JSZ Zone (SZDP) - Local rate limiting for low latency |
| **API Gateway Location** | JSZ Zone (SCP) - 2 nodes for HA |
| **Cross-Zone Communication** | Config sync + Analytics over secure connection |

---

## 2. Zone Details

### 2.1 Core Zone Components

| Component | Location | Port | Purpose |
|-----------|----------|------|---------|
| Console UI | Control Plane (K8s) | 443 | Management interface |
| Management API | Control Plane (K8s) | 8083 | REST API for configuration |
| Developer Portal | Control Plane (K8s) | 443 | API discovery |
| OpenAM SSO | Control Plane (K8s) | 443 | OAuth 2.0/OIDC provider |
| Elasticsearch | Control Plane (K8s) | 9200 | Analytics, logs, metrics |
| MongoDB | External to K8s | 27017 | API definitions, plans, subscriptions |
| Backend Services | Core Zone | Various | Microservices, legacy systems |

### 2.2 JSZ Zone - Secure Content Provider (SCP)

The SCP contains the API Gateway layer that handles all API traffic.

| Component | Instances | Port | Purpose |
|-----------|-----------|------|---------|
| Load Balancer | 1 | 443 | Traffic distribution (Active-Active) |
| API Gateway Node 1 | 2 pods | 8082 | API routing, security policies |
| API Gateway Node 2 | 2 pods | 8082 | API routing, security policies |

**Gateway Responsibilities:**
- JWT token validation
- Rate limiting (via Redis in SZDP)
- Request/Response transformation
- Caching
- Logging and analytics
- Circuit breaker
- IP filtering

### 2.3 JSZ Zone - Secure Zone Data Provider (SZDP)

The SZDP contains Redis for rate limiting and caching.

| Component | Instances | Port | Purpose |
|-----------|-----------|------|---------|
| Redis Master 1 | 1 + Replica | 6379 | Rate limiting, caching |
| Redis Master 2 | 1 + Replica | 6379 | Rate limiting, caching |
| Redis Master 3 | 1 + Replica | 6379 | Rate limiting, caching |

**Redis Functions:**
- Distributed rate limiting
- Quota management
- Response caching
- Session store
- Counter management

---

## 3. Communication Flows

### 3.1 Numbered Flow Reference

| # | Flow Name | Source | Destination | Protocol | Description |
|---|-----------|--------|-------------|----------|-------------|
| 1 | API Request | JSZ Consumers | Load Balancer (SCP) | HTTPS :443 | API consumers send requests |
| 2 | Rate Limit Check | Gateway (SCP) | Redis (SZDP) | TLS :6379 | Check rate limits and quotas |
| 3 | Config Sync | Gateway (SCP) | MongoDB (Core) | TLS :27017 | Pull API definitions (cross-zone) |
| 4 | Analytics Push | Gateway (SCP) | Elasticsearch (Core) | HTTPS :9200 | Push events and metrics (cross-zone) |
| 5 | JSZ Backend | Gateway (SCP) | JSZ Backend Services | HTTPS/mTLS | Route to Japan-specific APIs |
| 6 | Core Backend | Gateway (SCP) | Core Backend Services | HTTPS/mTLS | Route to Core Zone APIs (cross-zone) |

### 3.2 API Request Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           API REQUEST FLOW                                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   ┌──────────────┐                                                                  │
│   │ JSZ API      │                                                                  │
│   │ Consumer     │                                                                  │
│   └──────┬───────┘                                                                  │
│          │ 1. HTTPS Request with Bearer Token                                       │
│          ▼                                                                          │
│   ┌──────────────┐                                                                  │
│   │ Load Balancer│  (JSZ - SCP)                                                     │
│   └──────┬───────┘                                                                  │
│          │ 2. Route to available Gateway node                                       │
│          ▼                                                                          │
│   ┌──────────────┐         3. Check Rate Limit        ┌──────────────┐             │
│   │ API Gateway  │ ───────────────────────────────────►│ Redis        │             │
│   │ (JSZ - SCP)  │◄───────────────────────────────────│ (JSZ - SZDP) │             │
│   └──────┬───────┘         Rate Limit OK              └──────────────┘             │
│          │                                                                          │
│          │ 4. Validate JWT (cached JWKS or introspect)                             │
│          │                                                                          │
│          │ 5. Apply Policies (transform, cache, etc.)                              │
│          │                                                                          │
│          ▼                                                                          │
│   ┌──────────────┐                                                                  │
│   │ Backend      │  (JSZ or Core Zone)                                             │
│   │ Service      │                                                                  │
│   └──────┬───────┘                                                                  │
│          │ 6. Response                                                              │
│          ▼                                                                          │
│   ┌──────────────┐         7. Log Analytics           ┌──────────────┐             │
│   │ API Gateway  │ ───────────────────────────────────►│ Elasticsearch│             │
│   │ (JSZ - SCP)  │         (Async)                    │ (Core Zone)  │             │
│   └──────┬───────┘                                    └──────────────┘             │
│          │ 8. Return Response                                                       │
│          ▼                                                                          │
│   ┌──────────────┐                                                                  │
│   │ JSZ API      │                                                                  │
│   │ Consumer     │                                                                  │
│   └──────────────┘                                                                  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Configuration Sync Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        CONFIGURATION SYNC FLOW                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│         CORE ZONE                                    JSZ ZONE (SCP)                 │
│                                                                                      │
│   ┌──────────────┐                                                                  │
│   │ Admin User   │                                                                  │
│   └──────┬───────┘                                                                  │
│          │ 1. Create/Update API                                                     │
│          ▼                                                                          │
│   ┌──────────────┐                                                                  │
│   │ Console UI   │                                                                  │
│   └──────┬───────┘                                                                  │
│          │ 2. Save via Management API                                               │
│          ▼                                                                          │
│   ┌──────────────┐                                                                  │
│   │ Management   │                                                                  │
│   │ API          │                                                                  │
│   └──────┬───────┘                                                                  │
│          │ 3. Store in MongoDB                                                      │
│          ▼                                                                          │
│   ┌──────────────┐         4. Pull Config (Periodic)  ┌──────────────┐             │
│   │ MongoDB      │◄───────────────────────────────────│ Gateway      │             │
│   │ Cluster      │         TLS :27017                 │ Node 1       │             │
│   │              │         (Cross-Zone)               │ (JSZ - SCP)  │             │
│   └──────────────┘                                    └──────────────┘             │
│                                                                                      │
│   ┌──────────────┐         4. Pull Config (Periodic)  ┌──────────────┐             │
│   │ MongoDB      │◄───────────────────────────────────│ Gateway      │             │
│   │ Cluster      │         TLS :27017                 │ Node 2       │             │
│   │              │         (Cross-Zone)               │ (JSZ - SCP)  │             │
│   └──────────────┘                                    └──────────────┘             │
│                                                                                      │
│   Sync Interval: Every 5 seconds (configurable)                                     │
│   Data Synced: API Definitions, Plans, Subscriptions, Policies                      │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.4 Analytics Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           ANALYTICS FLOW                                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│         JSZ ZONE (SCP)                               CORE ZONE                      │
│                                                                                      │
│   ┌──────────────┐                                                                  │
│   │ Gateway      │                                                                  │
│   │ Node 1 & 2   │                                                                  │
│   └──────┬───────┘                                                                  │
│          │                                                                          │
│          │ Collect per-request:                                                     │
│          │ - Request/Response metadata                                              │
│          │ - Latency metrics                                                        │
│          │ - Error codes                                                            │
│          │ - Consumer/API/Plan info                                                 │
│          │                                                                          │
│          │ 1. Push Events (Async/Batch)                                             │
│          ▼                                                                          │
│   ┌──────────────┐         HTTPS :9200               ┌──────────────┐              │
│   │ Gateway      │ ──────────────────────────────────►│ Elasticsearch│              │
│   │ Reporter     │         (Cross-Zone)              │ (Core Zone)  │              │
│   └──────────────┘                                   └──────┬───────┘              │
│                                                             │                       │
│                                                             │ 2. Index & Store      │
│                                                             ▼                       │
│                                                      ┌──────────────┐              │
│                                                      │ Kibana /     │              │
│                                                      │ Grafana      │              │
│                                                      │ Dashboards   │              │
│                                                      └──────────────┘              │
│                                                                                      │
│   Events Pushed: Request logs, Response logs, Health metrics, Error events         │
│   Batch Size: 100 events (configurable)                                            │
│   Flush Interval: 5 seconds (configurable)                                         │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Network Configuration

### 4.1 Firewall Rules (JSZ → Core Zone)

A firewall exists between the JSZ Zone and Core Zone. The following rules are required:

| Rule | Source | Destination | Port | Protocol | Purpose |
|------|--------|-------------|------|----------|---------|
| FW-001 | JSZ Gateway (SCP) | Core MongoDB | 27017 | TLS | Config sync - Pull API definitions |
| FW-002 | JSZ Gateway (SCP) | Core Elasticsearch | 9200 | HTTPS | Analytics - Push events/metrics |
| FW-003 | JSZ Gateway (SCP) | Core Backends | 443 | HTTPS/mTLS | API forwarding to Core services |
| FW-004 | JSZ Gateway (SCP) | Core OpenAM | 443 | HTTPS | Token validation (JWKS/introspect) |

### 4.2 Cross-Zone Communication

| Source Zone | Destination Zone | Port | Protocol | Firewall | Purpose |
|-------------|------------------|------|----------|----------|---------|
| JSZ (SCP) | Core Zone | 27017 | TLS | Yes | MongoDB config sync |
| JSZ (SCP) | Core Zone | 9200 | HTTPS | Yes | Elasticsearch analytics |
| JSZ (SCP) | Core Zone | 443 | HTTPS/mTLS | Yes | Backend API calls |
| JSZ (SCP) | JSZ (SZDP) | 6379 | TLS | No | Redis rate limiting (within JSZ) |

### 4.3 Internal JSZ Communication (No Firewall)

| Source | Destination | Port | Protocol | Purpose |
|--------|-------------|------|----------|---------|
| Load Balancer | Gateway Node 1 | 8082 | HTTPS | API traffic |
| Load Balancer | Gateway Node 2 | 8082 | HTTPS | API traffic |
| Gateway Node 1 | Redis Cluster | 6379 | TLS | Rate limiting |
| Gateway Node 2 | Redis Cluster | 6379 | TLS | Rate limiting |
| Gateway Nodes | JSZ Backends | Various | HTTPS/mTLS | API forwarding |

### 4.4 Port Reference

```
CORE ZONE:
├── Console UI            : 443 (HTTPS)
├── Management API        : 8083 (HTTPS)
├── Developer Portal      : 443 (HTTPS)
├── OpenAM SSO           : 443 (HTTPS)
├── Elasticsearch        : 9200 (HTTPS)
├── MongoDB              : 27017 (TLS)
└── Backend Services     : Various

JSZ ZONE - SCP (Secure Content Provider):
├── Load Balancer        : 443 (HTTPS)
├── Gateway Node 1       : 8082 (HTTPS)
├── Gateway Node 2       : 8082 (HTTPS)
└── JSZ Backend Services : Various

JSZ ZONE - SZDP (Secure Zone Data Provider):
└── Redis Cluster        : 6379 (TLS)
```

---

## 5. Gateway Configuration

### 5.1 MongoDB Connection (Cross-Zone)

```yaml
# Gateway gravitee.yml - MongoDB configuration
management:
  type: mongodb
  mongodb:
    uri: mongodb://mongodb-primary.core-zone:27017,mongodb-secondary1.core-zone:27017,mongodb-secondary2.core-zone:27017/gravitee?replicaSet=rs0
    ssl:
      enabled: true
      truststore:
        path: /certs/mongodb-truststore.p12
        password: ${MONGODB_TRUSTSTORE_PASSWORD}
    connectTimeout: 5000
    socketTimeout: 30000
```

### 5.2 Redis Connection (SZDP)

```yaml
# Gateway gravitee.yml - Redis configuration for rate limiting
ratelimit:
  type: redis
  redis:
    host: redis-cluster.jsz-szdp
    port: 6379
    ssl: true
    password: ${REDIS_PASSWORD}
    
cache:
  type: redis
  redis:
    host: redis-cluster.jsz-szdp
    port: 6379
    ssl: true
    password: ${REDIS_PASSWORD}
```

### 5.3 Elasticsearch Connection (Cross-Zone)

```yaml
# Gateway gravitee.yml - Elasticsearch reporter
reporters:
  elasticsearch:
    enabled: true
    endpoints:
      - https://elasticsearch.core-zone:9200
    security:
      username: ${ES_USERNAME}
      password: ${ES_PASSWORD}
    index: gravitee
    bulk:
      actions: 100
      flush_interval: 5
```

---

## 6. High Availability

### 6.1 Component Redundancy

| Component | Zone | Replicas | HA Mode |
|-----------|------|----------|---------|
| Console UI | Core | 2 | Active-Active |
| Management API | Core | 2 | Active-Active |
| Developer Portal | Core | 2 | Active-Active |
| Elasticsearch | Core | 3 | Cluster |
| MongoDB | Core | 3 | Replica Set |
| API Gateway Node 1 | JSZ (SCP) | 2 pods | Active-Active |
| API Gateway Node 2 | JSZ (SCP) | 2 pods | Active-Active |
| Redis | JSZ (SZDP) | 6 | Cluster (3 masters + 3 replicas) |

### 6.2 Failover Scenarios

| Scenario | Impact | Recovery |
|----------|--------|----------|
| Gateway Node 1 fails | LB routes to Node 2 | Auto (immediate) |
| Gateway Node 2 fails | LB routes to Node 1 | Auto (immediate) |
| Redis Master fails | Replica promoted | Auto (< 30 seconds) |
| MongoDB Primary fails | Secondary promoted | Auto (< 30 seconds) |
| Elasticsearch node fails | Cluster rebalances | Auto (minutes) |

---

## 7. Security

### 7.1 Cross-Zone Security

- All cross-zone communication uses TLS 1.2/1.3
- Certificate-based authentication for MongoDB
- API key/password authentication for Elasticsearch
- Network isolation between zones (no direct user access to SZDP)

### 7.2 Gateway Security Policies

| Policy | Purpose |
|--------|---------|
| JWT Validation | Validate tokens from OpenAM (Core Zone) |
| Rate Limiting | Enforce via Redis (SZDP) |
| IP Filtering | Allow/deny by source IP |
| mTLS | Mutual TLS for backend calls |
| Request Throttling | Protect backends from overload |

---

## 8. PII/MNPI Data Protection

### 8.1 Data Residency Requirements

PII (Personally Identifiable Information) and MNPI (Material Non-Public Information) data must remain within the JSZ Zone and must NOT be transmitted to the Core Zone.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           DATA RESIDENCY MODEL                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   CORE ZONE (No PII/MNPI)              FIREWALL         JSZ ZONE (PII/MNPI Allowed) │
│   ┌─────────────────────┐             ║       ║        ┌─────────────────────────┐  │
│   │ Elasticsearch       │◄────────────╫───────╫────────│ Gateway (Filtered)      │  │
│   │ - API metrics only  │  Filtered   ║       ║        │ - Masks PII in logs     │  │
│   │ - No request body   │  Analytics  ║       ║        │ - Excludes body data    │  │
│   │ - No PII fields     │             ║       ║        │                         │  │
│   └─────────────────────┘             ║       ║        └─────────────────────────┘  │
│                                       ║       ║                    │                │
│   ┌─────────────────────┐             ║       ║        ┌───────────▼───────────────┐│
│   │ MongoDB             │             ║       ║        │ JSZ Local Database       ││
│   │ - API configs only  │             ║ BLOCK ║        │ - Customer PII           ││
│   │ - No customer data  │             ║  PII  ║        │ - Financial MNPI         ││
│   └─────────────────────┘             ║       ║        │ - Transaction records    ││
│                                       ║       ║        │ - Compliance audit logs  ││
│                                       ║       ║        └──────────────────────────┘│
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Data Classification

| Data Type | Zone | Can Leave JSZ? | Example |
|-----------|------|----------------|---------|
| **PII** | JSZ Only | NO | Names, SSN, Email, Phone, Address |
| **MNPI** | JSZ Only | NO | Trading data, Financial reports, Insider info |
| **API Metrics** | Both | YES | Request count, Latency, Status codes |
| **API Configs** | Core Zone | N/A | API definitions, Plans, Subscriptions |
| **Error Codes** | Both | YES | HTTP status, Error types (no details) |

### 8.3 Gateway Configuration for Data Filtering

#### 8.3.1 Exclude Request/Response Body from Analytics

```yaml
# gravitee.yml - Reporter configuration
reporters:
  elasticsearch:
    enabled: true
    endpoints:
      - https://elasticsearch.core-zone:9200
    security:
      username: ${ES_USERNAME}
      password: ${ES_PASSWORD}
    
    # CRITICAL: Exclude request/response content
    request:
      exclude:
        - body        # Never log request body (contains PII)
        - headers:
            - Authorization
            - X-Customer-ID
            - X-Account-Number
            - Cookie
            - Set-Cookie
    
    response:
      exclude:
        - body        # Never log response body (contains MNPI)
        - headers:
            - Set-Cookie
            - X-Customer-Token
    
    # Only include safe metadata
    request:
      include:
        - method
        - path
        - remote_address
        - content_length
    
    response:
      include:
        - status
        - content_length
        - response_time
```

#### 8.3.2 Mask Sensitive Fields in Logs

```yaml
# gravitee.yml - Logging configuration with masking
logging:
  mask:
    enabled: true
    patterns:
      # PII Patterns
      - pattern: '"ssn"\s*:\s*"[^"]*"'
        replacement: '"ssn":"***MASKED***"'
      - pattern: '"email"\s*:\s*"[^"]*"'
        replacement: '"email":"***MASKED***"'
      - pattern: '"phone"\s*:\s*"[^"]*"'
        replacement: '"phone":"***MASKED***"'
      - pattern: '"creditCard"\s*:\s*"[^"]*"'
        replacement: '"creditCard":"***MASKED***"'
      - pattern: '"accountNumber"\s*:\s*"[^"]*"'
        replacement: '"accountNumber":"***MASKED***"'
      
      # MNPI Patterns
      - pattern: '"balance"\s*:\s*"[^"]*"'
        replacement: '"balance":"***MASKED***"'
      - pattern: '"tradingData"\s*:\s*"[^"]*"'
        replacement: '"tradingData":"***MASKED***"'
```

#### 8.3.3 API-Level Logging Policy

```json
{
  "name": "Logging Policy - PII Safe",
  "policy": "logging",
  "configuration": {
    "scope": "REQUEST_RESPONSE",
    "content": "NONE",
    "condition": "",
    "maxSizeLogMessage": 0,
    "excludedResponseTypes": [
      "application/json",
      "application/xml"
    ]
  }
}
```

### 8.4 Transform Policy for PII Removal

Apply this policy to mask PII before analytics collection:

```json
{
  "name": "PII-Masking-Policy",
  "policy": "transform-headers",
  "configuration": {
    "removeHeaders": [
      "X-Customer-ID",
      "X-Account-Number",
      "X-SSN",
      "Authorization"
    ],
    "addHeaders": [
      {
        "name": "X-Request-Sanitized",
        "value": "true"
      }
    ]
  }
}
```

### 8.5 Reporter Filter Configuration

```yaml
# gravitee.yml - Advanced reporter filtering
reporters:
  elasticsearch:
    enabled: true
    
    # Filter what gets reported
    filter:
      # Only report these event types
      include:
        - REQUEST
        - RESPONSE
        - HEALTH_CHECK
      
      # Exclude specific APIs from detailed logging
      exclude:
        apis:
          - "api-with-pii-data"
          - "financial-transactions-api"
    
    # Field-level filtering
    fields:
      exclude:
        - "request.body"
        - "response.body"
        - "request.headers.authorization"
        - "request.headers.x-customer-*"
        - "client.user"
```

### 8.6 Analytics Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    ANALYTICS FLOW WITH PII FILTERING                                 │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   JSZ ZONE                                                                          │
│   ┌───────────────────────────────────────────────────────────────────────────┐    │
│   │                                                                            │    │
│   │   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐           │    │
│   │   │ API Request  │      │ Gateway      │      │ PII Filter   │           │    │
│   │   │ (with PII)   │─────►│ Processing   │─────►│ Policy       │           │    │
│   │   └──────────────┘      └──────────────┘      └──────┬───────┘           │    │
│   │                                                       │                   │    │
│   │                                              Filtered │ Analytics         │    │
│   │                                              (No PII) │                   │    │
│   │                                                       ▼                   │    │
│   │   ┌──────────────┐                           ┌──────────────┐            │    │
│   │   │ Backend      │◄──────────────────────────│ Route to     │            │    │
│   │   │ (PII stays)  │      Full Request         │ Backend      │            │    │
│   │   └──────────────┘      (with PII)           └──────────────┘            │    │
│   │                                                                            │    │
│   └───────────────────────────────────────────────────────────────────────────┘    │
│                                                       │                             │
│                                                       │ Filtered Analytics          │
│                                                       │ (metrics only)              │
│                                                       ▼                             │
│   ═══════════════════════════════════════════════════════════════════════════════  │
│                                    FIREWALL                                         │
│   ═══════════════════════════════════════════════════════════════════════════════  │
│                                                       │                             │
│                                                       ▼                             │
│   CORE ZONE                                                                         │
│   ┌───────────────────────────────────────────────────────────────────────────┐    │
│   │   Elasticsearch receives:                                                  │    │
│   │   ✓ Request method, path, status code                                     │    │
│   │   ✓ Response time, content length                                         │    │
│   │   ✓ API ID, Plan ID, Application ID                                       │    │
│   │   ✓ Error codes (no details)                                              │    │
│   │   ✗ NO request body                                                       │    │
│   │   ✗ NO response body                                                      │    │
│   │   ✗ NO PII headers                                                        │    │
│   │   ✗ NO customer identifiers                                               │    │
│   └───────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 8.7 Verification Checklist

| Check | Configuration | Verified |
|-------|---------------|----------|
| Request body excluded | `reporters.elasticsearch.request.exclude: body` | [ ] |
| Response body excluded | `reporters.elasticsearch.response.exclude: body` | [ ] |
| Auth headers excluded | `reporters.elasticsearch.request.exclude.headers` | [ ] |
| PII masking enabled | `logging.mask.enabled: true` | [ ] |
| Customer ID headers removed | Transform policy applied | [ ] |
| API-level logging disabled for PII APIs | Logging policy: content=NONE | [ ] |

### 8.8 Compliance Notes

| Regulation | Requirement | Configuration |
|------------|-------------|---------------|
| **GDPR** | PII must not leave EU without consent | Keep PII in JSZ, filter analytics |
| **CCPA** | Personal info must be protected | Mask PII in all logs |
| **SOX** | Financial data audit trail | Log in JSZ only, not Core Zone |
| **PCI-DSS** | Card data must be masked | Mask creditCard patterns |
| **Japan APPI** | Personal data residency | PII stays in JSZ Zone |

---

## 9. Summary

### Key Points

| Aspect | Configuration |
|--------|---------------|
| **MongoDB** | Core Zone (external to K8s) - Central config store |
| **Elasticsearch** | Core Zone (in-cluster) - Filtered analytics only |
| **Redis** | JSZ Zone (SZDP) - Local rate limiting |
| **API Gateway** | JSZ Zone (SCP) - 2 nodes, 2 pods each |
| **Cross-Zone Traffic** | Config sync (MongoDB) + Filtered Analytics (ES) |
| **Intra-Zone Traffic** | Gateway ↔ Redis for rate limiting |
| **PII/MNPI Data** | Stays in JSZ Zone only - Never sent to Core Zone |
| **Analytics Filtering** | Request/Response body excluded, PII headers masked |

---

## 10. Related Documents

| Document | Description |
|----------|-------------|
| [INTERNAL-GATEWAY-ARCHITECTURE.md](./INTERNAL-GATEWAY-ARCHITECTURE.md) | Core Zone only architecture |
| [TLS-OAUTH-CONFIGURATION.md](./TLS-OAUTH-CONFIGURATION.md) | Security configuration |
| [DEPLOYMENT-GUIDE.md](./DEPLOYMENT-GUIDE.md) | Kubernetes deployment |
| [PRE-DEPLOYMENT-CONNECTIVITY-CHECKLIST.md](./PRE-DEPLOYMENT-CONNECTIVITY-CHECKLIST.md) | Connectivity verification |

# Japan Secure Zone (JSZ) Architecture

## Overview

This document details the Japan Secure Zone (JSZ) architecture, which is **completely separate** from the global architecture. JSZ is an isolated environment designed for Japan-specific compliance requirements with internal-only API traffic.

![JSZ Architecture](./diagrams/JSZ-ARCHITECTURE.svg)

---

## ⚠️ Critical: JSZ is Completely Separate

> **JSZ does NOT follow the global architecture pattern.**
> 
> Key differences:
> - ⛔ **NO External API Gateway** - Internal traffic only
> - ✓ Internal Gateway cluster only (2 nodes)
> - ✓ Gateway directly syncs with Core Zone (no separate sync agent)
> - ✓ One-way connections (JSZ initiates all connections - outbound only)
> - ✓ Complete network isolation
> - ✓ Japan compliance and data sovereignty requirements

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    JAPAN SECURE ZONE (JSZ)                      │
│                 Isolated | Internal Only                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ⛔ NO EXTERNAL API GATEWAY                                    │
│   JSZ does not expose APIs to internet                          │
│                                                                  │
│   ┌───────────────────────────────────────────────────────┐    │
│   │           JSZ INTERNAL APPLICATIONS                    │    │
│   │   Finance | HR | Compliance | Internal Operations      │    │
│   └───────────────────────────────────────────────────────┘    │
│                              │                                   │
│                              ▼                                   │
│   ┌───────────────────────────────────────────────────────┐    │
│   │         JSZ INTERNAL API GATEWAY CLUSTER               │    │
│   │   ┌─────────────────┐   ┌─────────────────┐           │    │
│   │   │ Internal GW     │   │ Internal GW     │           │    │
│   │   │ JSZ-Node-1      │   │ JSZ-Node-2      │           │    │
│   │   │ (Active)        │   │ (Active)        │           │    │
│   │   │                 │   │                 │           │    │
│   │   │ ↓ Pulls Config  │   │ ↑ Pushes        │           │    │
│   │   │ ↑ Pushes Analyt.│   │   Analytics     │           │    │
│   │   └─────────────────┘   └─────────────────┘           │    │
│   │                                                        │    │
│   │   2 Nodes | Active-Active | Direct Core Zone Sync     │    │
│   └───────────────────────────────────────────────────────┘    │
│                              │                                   │
│                              ▼                                   │
│   ┌───────────────────────────────────────────────────────┐    │
│   │           JSZ APP ZONE (Backend Services)              │    │
│   │   Japan-specific microservices | Databases             │    │
│   └───────────────────────────────────────────────────────┘    │
│                              │                                   │
└──────────────────────────────┼───────────────────────────────────┘
                               │ (Outbound Only)
                    ═══════════╪═══════════  FIREWALL
                               │
                               ▼
                  ┌──────────────────────────┐
                  │        CORE ZONE         │
                  │   MySQL | Elasticsearch  │
                  │   (Gateway connects      │
                  │    directly)             │
                  └──────────────────────────┘
```

---

## 2. Why JSZ is Different

| Aspect | Global Regions (US/EU/ASIA) | Japan Secure Zone (JSZ) |
|--------|----------------------------|-------------------------|
| External Gateway | ✓ Yes (public APIs) | ⛔ No |
| Internal Gateway | ✓ Yes (Core Zone) | ✓ Yes (JSZ isolated) |
| Internet Exposure | ✓ Yes | ⛔ No |
| Network Connectivity | Bidirectional with Core | One-way (outbound only) |
| Data Residency | Shared Core Zone | JSZ data stays in JSZ |
| Config Source | Direct from Core Zone | Gateway pulls from Core |
| Analytics Destination | Direct to Core ES | Gateway pushes to Core |
| Sync Method | Direct connection | Gateway direct sync |

---

## 3. Component Details

### 3.1 JSZ Internal Applications

| Category | Examples |
|----------|----------|
| Finance | Accounting systems, payment processing |
| HR | Employee management, payroll |
| Compliance | Audit systems, regulatory reporting |
| Operations | Internal tools, automation systems |

### 3.2 JSZ Internal API Gateway Cluster

| Property | Value |
|----------|-------|
| Nodes | 2 |
| Mode | Active-Active |
| HA | Enabled |
| Location | JSZ Data Center |
| Purpose | Internal app-to-app communication |
| Sync Method | **Gateway directly connects to Core Zone** |

**Gateway Configuration:**
```yaml
deployment:
  type: standalone (ZIP)
  nodes: 2
  mode: active-active
  
network:
  http_port: 8082
  mtls: enabled
  external_access: disabled
  
# Gateway pulls config directly from Core Zone MySQL
management:
  type: jdbc
  jdbc:
    url: jdbc:mysql://core-zone-mysql:3306/gravitee
      
# Gateway pushes analytics directly to Core Zone Elasticsearch
reporters:
  elasticsearch:
    endpoints:
      - https://core-zone-elasticsearch:9200
```

---

## 4. Data Flow

### 4.1 Configuration Sync Flow (Gateway Direct Pull)

The Gateway server itself pulls configuration directly from Core Zone:

```
Core Zone MySQL (API Configs)
         │
         │ (JSZ Gateway pulls directly)
         ▼
   JSZ Internal GW Cluster
   (Node-1, Node-2)
         │
         ▼
   API Definitions Applied
```

**Sync Details:**
| Parameter | Value |
|-----------|-------|
| Sync Interval | 5-10 seconds (configurable) |
| Protocol | HTTPS with mTLS |
| Connection | Gateway → Core Zone MySQL (Port 3306) |

### 4.2 Analytics Flow (Gateway Direct Push)

The Gateway server itself pushes analytics directly to Core Zone:

```
JSZ Internal Applications
         │
         │ (API calls)
         ▼
   JSZ Internal GW Cluster
         │
         │ (Gateway pushes directly)
         ▼
   Core Zone Elasticsearch
```

**Analytics Push Details:**
| Parameter | Value |
|-----------|-------|
| Push Mode | Real-time or batched |
| Protocol | HTTPS with mTLS |
| Connection | Gateway → Core Zone ES (Port 9200) |
| Data Anonymization | Enabled for sensitive fields |

### 4.3 Internal API Traffic Flow

```
JSZ App A                JSZ App B
    │                        ▲
    │ API Request            │ Response
    ▼                        │
┌─────────────────────────────────────┐
│     JSZ Internal Gateway Cluster    │
│  ┌───────────┐   ┌───────────┐     │
│  │ JSZ-Node-1│   │ JSZ-Node-2│     │
│  └───────────┘   └───────────┘     │
│                                     │
│  • Authentication (mTLS, JWT)       │
│  • Rate Limiting                    │
│  • Routing                          │
│  • Analytics Collection             │
│  • Direct sync with Core Zone       │
└─────────────────────────────────────┘
```

---

## 5. Security Architecture

### 5.1 Network Isolation

| Rule | Description |
|------|-------------|
| No Inbound from Internet | JSZ has no external exposure |
| No Inbound from Core Zone | Core Zone cannot initiate to JSZ |
| Outbound to Core Zone Only | Gateway connects directly to Core Zone |
| mTLS Required | All outbound connections use mTLS |

### 5.2 Firewall Rules

#### Inbound (Blocked)

| Source | Destination | Port | Action |
|--------|-------------|------|--------|
| Internet | JSZ | Any | **DENY** |
| Core Zone | JSZ | Any | **DENY** |
| Other Regions | JSZ | Any | **DENY** |

#### Outbound (Gateway Direct Connections)

| Source | Destination | Port | Protocol | Action |
|--------|-------------|------|----------|--------|
| JSZ Gateway | Core Zone MySQL | 3306 | TCP/mTLS | ALLOW |
| JSZ Gateway | Core Zone ES | 9200 | HTTPS/mTLS | ALLOW |
| JSZ | Any other | Any | Any | **DENY** |

#### Internal JSZ Traffic

| Source | Destination | Port | Protocol | Action |
|--------|-------------|------|----------|--------|
| JSZ Apps | JSZ Internal GW | 8082 | HTTP/mTLS | ALLOW |
| JSZ Internal GW | JSZ Backends | App Ports | HTTPS | ALLOW |

### 5.3 Authentication

| Method | Use Case |
|--------|----------|
| mTLS | Gateway to Core Zone connections |
| JWT | Internal application authentication |
| Internal API Key | Simple internal integrations |

---

## 6. Gateway Configuration Example

```yaml
# gravitee.yml for JSZ Gateway

############################################
# Gateway Settings
############################################
http:
  port: 8082
  secured: true
  ssl:
    clientAuth: required
    keystore:
      type: PKCS12
      path: /path/to/jsz-gateway-keystore.p12
    truststore:
      type: PKCS12
      path: /path/to/jsz-gateway-truststore.p12

############################################
# Management Repository (Pull Config from Core Zone)
############################################
management:
  type: jdbc

ds:
  mysql:
    host: core-zone-mysql.corp.internal
    port: 3306
    database: gravitee
    username: ${MYSQL_USER}
    password: ${MYSQL_PASSWORD}
    # mTLS settings
    sslEnabled: true
    sslMode: VERIFY_IDENTITY

############################################
# Analytics Reporter (Push to Core Zone ES)
############################################
reporters:
  elasticsearch:
    enabled: true
    endpoints:
      - https://core-zone-elasticsearch.corp.internal:9200
    security:
      username: ${ES_USER}
      password: ${ES_PASSWORD}
    ssl:
      keystore:
        type: PKCS12
        path: /path/to/jsz-gateway-keystore.p12

############################################
# Rate Limit Repository
############################################
ratelimit:
  type: jdbc
  jdbc:
    url: jdbc:mysql://core-zone-mysql.corp.internal:3306/gravitee
```

---

## 7. Compliance & Data Sovereignty

### 7.1 Japan-Specific Requirements

| Requirement | Implementation |
|-------------|----------------|
| Data Residency | JSZ data stays in Japan data center |
| Access Control | Japan-based administrators only |
| Audit Logging | All access logged with 7-year retention |
| Encryption | AES-256 at rest, TLS 1.3 in transit |

### 7.2 What Stays in JSZ

- ✓ Request/response payloads
- ✓ User PII (Personally Identifiable Information)
- ✓ Business-sensitive data
- ✓ Local application logs

### 7.3 What Goes to Core Zone

- ✓ Anonymized analytics (counts, latencies)
- ✓ Error aggregates (no sensitive details)
- ✗ No PII
- ✗ No request/response bodies

---

## 8. High Availability

### 8.1 Internal Gateway HA

| Property | Configuration |
|----------|---------------|
| Nodes | 2 (Active-Active) |
| Health Check | Every 5 seconds |
| Failover | Automatic within cluster |
| Load Balancing | Round-robin |

### 8.2 Core Zone Connectivity HA

If Core Zone connection fails:
1. JSZ Gateway continues operating with cached configuration
2. Config cache TTL: 24 hours
3. Analytics buffered locally (configurable buffer size)
4. Automatic reconnection with exponential backoff
5. Alerts triggered for operations team

---

## 9. Monitoring

### 9.1 Local Monitoring (JSZ)

| Metric | Description |
|--------|-------------|
| Gateway Health | Node availability, CPU, memory |
| Request Rate | Internal API requests per second |
| Error Rate | 4xx/5xx responses |
| Core Zone Sync Status | Last successful sync time |
| Connection Health | MySQL/ES connection status |

### 9.2 Alerts

| Alert | Condition | Severity |
|-------|-----------|----------|
| Core Zone Unreachable | Connection failed > 5 min | Critical |
| Config Sync Stale | Last sync > 1 hour | Warning |
| Analytics Buffer Full | Buffer > 80% | Warning |
| Gateway Node Down | Node health check failed | Critical |

---

## 10. Operations

### 10.1 Configuration Changes

1. Changes made in Core Zone Console
2. Published to Core Zone MySQL
3. JSZ Gateway pulls changes (within sync interval)
4. API definitions automatically updated

### 10.2 Gateway Updates

1. Rolling update (one node at a time)
2. Drain connections from node
3. Update gateway binary
4. Restart and health check
5. Move to next node

### 10.3 Emergency Isolation

If JSZ needs to be completely isolated:
1. Block all outbound firewall rules
2. JSZ continues with cached config
3. Analytics buffered locally
4. Manual config export/import when reconnected

---

## 11. Summary

| Aspect | JSZ Configuration |
|--------|-------------------|
| External Gateway | ⛔ **None** |
| Internal Gateway | ✓ 2 nodes (Active-Active) |
| Network | Outbound only to Core Zone |
| Sync Method | **Gateway direct sync** (no separate agent) |
| Config Pull | Gateway → Core Zone MySQL |
| Analytics Push | Gateway → Core Zone Elasticsearch |
| Data Sovereignty | JSZ data stays in JSZ |
| Compliance | Japan regulations enforced |

---

## 12. Related Documents

| Document | Description |
|----------|-------------|
| [Global Architecture](./GLOBAL-COMPLETE-ARCHITECTURE.md) | Global view (excludes JSZ) |
| [External Gateway Architecture](./EXTERNAL-GATEWAY-ARCHITECTURE.md) | External gateway details (N/A for JSZ) |
| [Internal Gateway Architecture](./INTERNAL-GATEWAY-ARCHITECTURE.md) | Core Zone internal gateways |
| [Firewall Rules](./FIREWALL_RULES_MULTI_REGION.md) | Complete firewall configurations |

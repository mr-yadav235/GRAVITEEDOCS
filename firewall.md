# Firewall Requirements Document

## Overview

This document defines the firewall rules required for the Internal API Gateway architecture with OpenAM SSO in the Inner DMZ.

---

## Zone Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              INNER DMZ                                       │
│                         (Authentication Only)                                │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      OpenAM SSO Server                               │   │
│   │                   sso.inner-dmz.company.com                          │   │
│   │                        Port: 443 (HTTPS)                             │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │       Firewall Zone A       │
                    └──────────────┬──────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────────────┐
│                              CORE ZONE                                       │
│                    (Gravitee Platform + Data Services)                       │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  Control Plane          │  Data Plane        │  Data Services       │   │
│   │  • Console UI (:8080)   │  • US Gateway      │  • MongoDB (:27017)  │   │
│   │  • Portal (:8080)       │  • EU Gateway      │  • Redis (:6379)     │   │
│   │  • Mgmt API (:8083)     │  • APAC Gateway    │  • ES (:9200)        │   │
│   │                         │  (All on :8082)    │                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │       Firewall Zone B       │
                    └──────────────┬──────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────────────┐
│                           CORPORATE ZONE                                     │
│                         (Admin Users / Developers)                           │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  👤 API Developers    👤 Platform Admins    👨‍💻 App Developers       │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Firewall Rules Matrix

### 1.1 Corporate Zone → Inner DMZ

| Rule ID | Source | Destination | Port | Protocol | Direction | Purpose |
|---------|--------|-------------|------|----------|-----------|---------|
| FW-001 | Corporate Zone | OpenAM SSO | 443 | HTTPS | Inbound | User authentication (OAuth 2.0 redirect) |

### 1.2 Corporate Zone → Core Zone

| Rule ID | Source | Destination | Port | Protocol | Direction | Purpose |
|---------|--------|-------------|------|----------|-----------|---------|
| FW-002 | Corporate Zone | Console UI | 443 | HTTPS | Inbound | Admin access to Console |
| FW-003 | Corporate Zone | Developer Portal | 443 | HTTPS | Inbound | Developer access to Portal |
| FW-004 | Corporate Zone | Management API | 443 | HTTPS | Inbound | API management (optional direct) |

### 1.3 Core Zone → Inner DMZ

| Rule ID | Source | Destination | Port | Protocol | Direction | Purpose |
|---------|--------|-------------|------|----------|-----------|---------|
| FW-005 | Console UI | OpenAM SSO | 443 | HTTPS | Outbound | Console login (OAuth 2.0) |
| FW-006 | Developer Portal | OpenAM SSO | 443 | HTTPS | Outbound | Portal login (OAuth 2.0) |
| FW-007 | Management API | OpenAM SSO | 443 | HTTPS | Outbound | Token validation |
| FW-008 | API Gateway (US) | OpenAM SSO | 443 | HTTPS | Outbound | JWT/Token validation |
| FW-009 | API Gateway (EU) | OpenAM SSO | 443 | HTTPS | Outbound | JWT/Token validation |
| FW-010 | API Gateway (APAC) | OpenAM SSO | 443 | HTTPS | Outbound | JWT/Token validation |

### 1.4 Core Zone Internal (Data Plane)

| Rule ID | Source | Destination | Port | Protocol | Direction | Purpose |
|---------|--------|-------------|------|----------|-----------|---------|
| FW-011 | API Consumers | Global LB | 443 | HTTPS | Inbound | API traffic entry point |
| FW-012 | Global LB | Regional LB (US) | 443 | HTTPS | Internal | Geo-routing |
| FW-013 | Global LB | Regional LB (EU) | 443 | HTTPS | Internal | Geo-routing |
| FW-014 | Global LB | Regional LB (APAC) | 443 | HTTPS | Internal | Geo-routing |
| FW-015 | Regional LB (US) | Gateway (US) | 8082 | HTTP | Internal | Traffic to gateway pods |
| FW-016 | Regional LB (EU) | Gateway (EU) | 8082 | HTTP | Internal | Traffic to gateway pods |
| FW-017 | Regional LB (APAC) | Gateway (APAC) | 8082 | HTTP | Internal | Traffic to gateway pods |
| FW-018 | Gateway (All) | Backend APIs | 8080 | HTTP/HTTPS | Outbound | Route to microservices |

### 1.5 Core Zone Internal (Control Plane)

| Rule ID | Source | Destination | Port | Protocol | Direction | Purpose |
|---------|--------|-------------|------|----------|-----------|---------|
| FW-019 | Console UI | Management API | 8083 | HTTP | Internal | UI to API calls |
| FW-020 | Developer Portal | Management API | 8083 | HTTP | Internal | Portal to API calls |
| FW-021 | Management API | MongoDB | 27017 | TCP | Internal | Config storage |
| FW-022 | Console UI | MongoDB | 27017 | TCP | Internal | Direct queries (optional) |

### 1.6 Core Zone Internal (Data Services)

| Rule ID | Source | Destination | Port | Protocol | Direction | Purpose |
|---------|--------|-------------|------|----------|-----------|---------|
| FW-023 | Gateway (All) | MongoDB | 27017 | TCP | Internal | Config sync (pull) |
| FW-024 | Gateway (All) | Redis | 6379 | TCP | Internal | Rate limiting / caching |
| FW-025 | Gateway (All) | Elasticsearch | 9200 | TCP | Internal | Analytics push |
| FW-026 | Management API | Redis | 6379 | TCP | Internal | Cache management |
| FW-027 | Management API | Elasticsearch | 9200 | TCP | Internal | Analytics queries |

### 1.7 Inner DMZ Internal

| Rule ID | Source | Destination | Port | Protocol | Direction | Purpose |
|---------|--------|-------------|------|----------|-----------|---------|
| FW-028 | OpenAM SSO | LDAP/AD | 636 | LDAPS | Outbound | User directory lookup |
| FW-029 | OpenAM SSO | LDAP/AD | 389 | LDAP | Outbound | User directory lookup (if no TLS) |

---

## 2. Detailed Firewall Rules

### 2.1 OpenAM SSO Server (Inner DMZ)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          OpenAM SSO FIREWALL RULES                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  INBOUND:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Source          │ Port │ Protocol │ Purpose                        │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │ Corporate Zone  │ 443  │ HTTPS    │ User login (browser redirect)  │   │
│  │ Core Zone       │ 443  │ HTTPS    │ Token validation / JWKS        │   │
│  │ Console UI      │ 443  │ HTTPS    │ OAuth 2.0 code exchange        │   │
│  │ Portal UI       │ 443  │ HTTPS    │ OAuth 2.0 code exchange        │   │
│  │ API Gateway     │ 443  │ HTTPS    │ Token introspection / JWKS     │   │
│  │ Management API  │ 443  │ HTTPS    │ Token validation               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  OUTBOUND:                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Destination     │ Port │ Protocol │ Purpose                        │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │ LDAP/AD         │ 636  │ LDAPS    │ User authentication            │   │
│  │ LDAP/AD         │ 389  │ LDAP     │ User authentication (non-TLS)  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 API Gateway (Core Zone)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        API GATEWAY FIREWALL RULES                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  INBOUND:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Source          │ Port │ Protocol │ Purpose                        │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │ Regional LB     │ 8082 │ HTTP     │ API traffic from load balancer │   │
│  │ K8s Ingress     │ 8082 │ HTTP     │ K8s service routing            │   │
│  │ Health Checks   │ 18082│ HTTP     │ Gateway health endpoint        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  OUTBOUND:                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Destination     │ Port │ Protocol │ Purpose                        │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │ OpenAM (DMZ)    │ 443  │ HTTPS    │ Token validation / JWKS        │   │
│  │ MongoDB         │ 27017│ TCP      │ Config sync                    │   │
│  │ Redis           │ 6379 │ TCP      │ Rate limiting / caching        │   │
│  │ Elasticsearch   │ 9200 │ TCP      │ Analytics                      │   │
│  │ Backend APIs    │ *    │ HTTP/S   │ Route to microservices         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Control Plane Components (Core Zone)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      CONTROL PLANE FIREWALL RULES                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CONSOLE UI (Port 8080):                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ INBOUND:  Corporate Zone → Console UI (443/HTTPS via Ingress)       │   │
│  │ OUTBOUND: Console UI → OpenAM (443/HTTPS) - OAuth 2.0               │   │
│  │ OUTBOUND: Console UI → Management API (8083/HTTP)                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DEVELOPER PORTAL (Port 8080):                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ INBOUND:  Corporate Zone → Portal (443/HTTPS via Ingress)           │   │
│  │ OUTBOUND: Portal → OpenAM (443/HTTPS) - OAuth 2.0                   │   │
│  │ OUTBOUND: Portal → Management API (8083/HTTP)                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MANAGEMENT API (Port 8083):                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ INBOUND:  Console UI → Management API (8083/HTTP)                   │   │
│  │ INBOUND:  Portal → Management API (8083/HTTP)                       │   │
│  │ OUTBOUND: Management API → OpenAM (443/HTTPS) - Token validation    │   │
│  │ OUTBOUND: Management API → MongoDB (27017/TCP)                      │   │
│  │ OUTBOUND: Management API → Redis (6379/TCP)                         │   │
│  │ OUTBOUND: Management API → Elasticsearch (9200/TCP)                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Port Summary

### 3.1 All Required Ports

| Port | Protocol | Service | Zone | Description |
|------|----------|---------|------|-------------|
| **443** | HTTPS | OpenAM SSO | Inner DMZ | OAuth 2.0 / OIDC endpoints |
| **443** | HTTPS | Console UI (Ingress) | Core Zone | Admin web interface |
| **443** | HTTPS | Developer Portal (Ingress) | Core Zone | Developer web interface |
| **443** | HTTPS | Global LB | Core Zone | API traffic entry |
| **8080** | HTTP | Console UI (Pod) | Core Zone | Console application |
| **8080** | HTTP | Developer Portal (Pod) | Core Zone | Portal application |
| **8082** | HTTP | API Gateway (Pod) | Core Zone | Gateway API traffic |
| **8083** | HTTP | Management API (Pod) | Core Zone | Management REST API |
| **18082** | HTTP | API Gateway Technical | Core Zone | Health checks, metrics |
| **18083** | HTTP | Management API Technical | Core Zone | Health checks, metrics |
| **27017** | TCP | MongoDB | Core Zone | Configuration database |
| **6379** | TCP | Redis | Core Zone | Rate limiting, caching |
| **9200** | TCP | Elasticsearch | Core Zone | Analytics, logging |
| **636** | LDAPS | LDAP/AD | Corporate | User directory (secure) |
| **389** | LDAP | LDAP/AD | Corporate | User directory (non-secure) |

### 3.2 External Facing Ports (Through Ingress/LB)

| External Port | Internal Service | Description |
|---------------|------------------|-------------|
| 443 | Console UI (:8080) | `console.internal.company.com` |
| 443 | Developer Portal (:8080) | `portal.internal.company.com` |
| 443 | API Gateway (:8082) | `api.internal.company.com` |
| 443 | Management API (:8083) | `api-mgmt.internal.company.com` |
| 443 | OpenAM SSO | `sso.inner-dmz.company.com` |

---

## 4. Network Security Policies (Kubernetes)

### 4.1 API Gateway NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: gravitee-gateway-network-policy
  namespace: gravitee-data-plane
spec:
  podSelector:
    matchLabels:
      app: gravitee-gateway
  policyTypes:
    - Ingress
    - Egress
  
  ingress:
    # Allow from Regional Load Balancer / Ingress
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8082
    # Allow health checks
    - from:
        - namespaceSelector: {}
      ports:
        - protocol: TCP
          port: 18082
  
  egress:
    # Allow to OpenAM (Inner DMZ) for token validation
    - to:
        - ipBlock:
            cidr: 10.100.0.0/24  # Inner DMZ CIDR
      ports:
        - protocol: TCP
          port: 443
    # Allow to MongoDB
    - to:
        - namespaceSelector:
            matchLabels:
              name: gravitee-data
      ports:
        - protocol: TCP
          port: 27017
    # Allow to Redis
    - to:
        - namespaceSelector:
            matchLabels:
              name: gravitee-data
      ports:
        - protocol: TCP
          port: 6379
    # Allow to Elasticsearch
    - to:
        - namespaceSelector:
            matchLabels:
              name: gravitee-data
      ports:
        - protocol: TCP
          port: 9200
    # Allow to Backend APIs (Core Zone)
    - to:
        - namespaceSelector:
            matchLabels:
              zone: core
      ports:
        - protocol: TCP
          port: 8080
    # Allow DNS
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
```

### 4.2 Control Plane NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: gravitee-control-plane-network-policy
  namespace: gravitee-control-plane
spec:
  podSelector:
    matchLabels:
      tier: control-plane
  policyTypes:
    - Ingress
    - Egress
  
  ingress:
    # Allow from Corporate Zone (via Ingress)
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080  # Console UI, Portal
        - protocol: TCP
          port: 8083  # Management API
  
  egress:
    # Allow to OpenAM (Inner DMZ)
    - to:
        - ipBlock:
            cidr: 10.100.0.0/24  # Inner DMZ CIDR
      ports:
        - protocol: TCP
          port: 443
    # Allow to MongoDB
    - to:
        - namespaceSelector:
            matchLabels:
              name: gravitee-data
      ports:
        - protocol: TCP
          port: 27017
    # Allow to Redis
    - to:
        - namespaceSelector:
            matchLabels:
              name: gravitee-data
      ports:
        - protocol: TCP
          port: 6379
    # Allow to Elasticsearch
    - to:
        - namespaceSelector:
            matchLabels:
              name: gravitee-data
      ports:
        - protocol: TCP
          port: 9200
    # Allow DNS
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
```

---

## 5. Firewall Rule Request Template

### Rule Request Form

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      FIREWALL RULE REQUEST                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Request ID: ____________    Date: ____________    Requestor: ____________  │
│                                                                             │
│  Rule Details:                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Source Zone:        [  ] Inner DMZ  [  ] Core Zone  [  ] Corporate  │   │
│  │ Source IP/CIDR:     _______________________________________________  │   │
│  │ Source Service:     _______________________________________________  │   │
│  │                                                                      │   │
│  │ Destination Zone:   [  ] Inner DMZ  [  ] Core Zone  [  ] Corporate  │   │
│  │ Destination IP/CIDR:_______________________________________________  │   │
│  │ Destination Service:_______________________________________________  │   │
│  │                                                                      │   │
│  │ Port(s):           _______________________________________________   │   │
│  │ Protocol:          [  ] TCP  [  ] UDP  [  ] HTTPS  [  ] HTTP        │   │
│  │ Direction:         [  ] Inbound  [  ] Outbound  [  ] Bidirectional  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Business Justification:                                                    │
│  _______________________________________________________________________   │
│  _______________________________________________________________________   │
│                                                                             │
│  Approval: ____________    Date: ____________    Signature: ____________   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Security Considerations

### 6.1 Encryption Requirements

| Connection | Encryption | Certificate |
|------------|------------|-------------|
| Corporate → OpenAM | TLS 1.2+ | Valid CA-signed |
| Corporate → Console/Portal | TLS 1.2+ | Valid CA-signed |
| Core Zone → OpenAM | TLS 1.2+ | Valid CA-signed |
| Gateway → Backend | TLS 1.2+ or mTLS | Internal CA |
| Internal K8s traffic | Optional (service mesh) | Internal CA |

### 6.2 Default Deny Policy

- **All zones should implement default DENY**
- Only explicitly allowed traffic should pass
- Log all denied traffic for security monitoring

### 6.3 IP Whitelisting

For additional security, consider whitelisting specific IP ranges:

```yaml
# Example: Only allow specific Corporate Zone IPs
allowedSourceCIDRs:
  - 10.50.0.0/16    # Corporate Zone Network
  - 10.51.0.0/16    # VPN Users
```

---

## 7. Monitoring & Logging

### 7.1 Firewall Logs to Collect

| Log Type | Retention | Purpose |
|----------|-----------|---------|
| Allowed Traffic | 30 days | Audit trail |
| Denied Traffic | 90 days | Security analysis |
| Authentication Events | 1 year | Compliance |

### 7.2 Alerting Rules

| Alert | Condition | Severity |
|-------|-----------|----------|
| Unusual Denied Traffic | > 100 denies/min from single IP | High |
| OpenAM Connection Failure | Gateway cannot reach OpenAM | Critical |
| Port Scan Detected | Multiple port attempts | High |

---

## Related Documents

| Document | Description |
|----------|-------------|
| [Architecture Overview](./HLD-CORE-ZONE-ARCHITECTURE.md) | System architecture |
| [Deployment Guide](./DEPLOYMENT-GUIDE.md) | Installation steps |
| [User Interaction Flows](./USER-INTERACTION-FLOWS.md) | Authentication flows |

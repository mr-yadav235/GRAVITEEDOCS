# TLS Profiles & OAuth Configuration Guide

## Overview

This document details the TLS (Transport Layer Security) profiles and OAuth 2.0 configuration required for secure communication between Gravitee APIM components and external systems.

---

## 1. Component Communication Matrix

### 1.1 Internal Component Communication

| Source | Destination | Protocol | Port | TLS Required |
|--------|-------------|----------|------|--------------|
| Console UI | Management API | HTTPS | 8083 | Yes |
| Developer Portal | Management API | HTTPS | 8083 | Yes |
| API Gateway | Management API | HTTPS | 8083 | Yes |
| API Gateway | MongoDB | TLS | 27017 | Yes |
| API Gateway | Redis | TLS | 6379 | Yes |
| API Gateway | Elasticsearch | HTTPS | 9200 | Yes |
| Management API | MongoDB | TLS | 27017 | Yes |
| Management API | Elasticsearch | HTTPS | 9200 | Yes |
| Management API | OpenAM SSO | HTTPS | 443 | Yes |
| Console UI | OpenAM SSO | HTTPS | 443 | Yes |
| Developer Portal | OpenAM SSO | HTTPS | 443 | Yes |
| API Gateway | OpenAM SSO | HTTPS | 443 | Yes |
| API Gateway | Backend APIs | HTTPS | Various | Yes (mTLS optional) |

### 1.2 External Communication

| Source | Destination | Protocol | Port | TLS Required |
|--------|-------------|----------|------|--------------|
| Corporate Users | Console UI | HTTPS | 443 | Yes |
| Corporate Users | Developer Portal | HTTPS | 443 | Yes |
| API Consumers | API Gateway | HTTPS | 443 | Yes |
| Corporate Users | OpenAM SSO | HTTPS | 443 | Yes |

---

## 2. TLS Profiles Configuration

### 2.1 Certificate Requirements

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CERTIFICATE HIERARCHY                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│                    ┌─────────────────────┐                              │
│                    │   Root CA           │                              │
│                    │   (Enterprise PKI)  │                              │
│                    └──────────┬──────────┘                              │
│                               │                                          │
│              ┌────────────────┼────────────────┐                        │
│              ▼                ▼                ▼                        │
│    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐         │
│    │ Intermediate CA │ │ Intermediate CA │ │ Intermediate CA │         │
│    │ (Internal)      │ │ (DMZ)           │ │ (External)      │         │
│    └────────┬────────┘ └────────┬────────┘ └────────┬────────┘         │
│             │                   │                   │                   │
│             ▼                   ▼                   ▼                   │
│    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐         │
│    │ Server Certs    │ │ Server Certs    │ │ Server Certs    │         │
│    │ • Console UI    │ │ • OpenAM SSO    │ │ • Public LB     │         │
│    │ • Mgmt API      │ │                 │ │                 │         │
│    │ • Portal        │ │                 │ │                 │         │
│    │ • Gateway       │ │                 │ │                 │         │
│    │ • MongoDB       │ │                 │ │                 │         │
│    │ • Redis         │ │                 │ │                 │         │
│    │ • Elasticsearch │ │                 │ │                 │         │
│    └─────────────────┘ └─────────────────┘ └─────────────────┘         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Certificate Specifications

| Certificate | Type | Key Size | Validity | SAN (Subject Alternative Names) |
|-------------|------|----------|----------|--------------------------------|
| Console UI | Server | RSA 2048+ / ECDSA P-256 | 1 year | console.core.company.com |
| Management API | Server | RSA 2048+ / ECDSA P-256 | 1 year | api.core.company.com |
| Developer Portal | Server | RSA 2048+ / ECDSA P-256 | 1 year | portal.core.company.com |
| API Gateway | Server | RSA 2048+ / ECDSA P-256 | 1 year | gateway.core.company.com, *.gateway.core.company.com |
| MongoDB | Server | RSA 2048+ / ECDSA P-256 | 1 year | mongo-0.core.company.com, mongo-1.core.company.com, mongo-2.core.company.com |
| Redis | Server | RSA 2048+ / ECDSA P-256 | 1 year | redis.core.company.com |
| Elasticsearch | Server | RSA 2048+ / ECDSA P-256 | 1 year | es-0.core.company.com, es-1.core.company.com, es-2.core.company.com |

### 2.3 Kubernetes TLS Secrets

#### Create TLS Secrets for Each Component

```bash
# Create namespace
kubectl create namespace gravitee

# Console UI TLS Secret
kubectl create secret tls console-tls \
  --cert=/path/to/console.crt \
  --key=/path/to/console.key \
  -n gravitee

# Management API TLS Secret
kubectl create secret tls management-api-tls \
  --cert=/path/to/management-api.crt \
  --key=/path/to/management-api.key \
  -n gravitee

# Developer Portal TLS Secret
kubectl create secret tls portal-tls \
  --cert=/path/to/portal.crt \
  --key=/path/to/portal.key \
  -n gravitee

# API Gateway TLS Secret
kubectl create secret tls gateway-tls \
  --cert=/path/to/gateway.crt \
  --key=/path/to/gateway.key \
  -n gravitee

# CA Bundle Secret (for trusting internal CA)
kubectl create secret generic ca-bundle \
  --from-file=ca.crt=/path/to/ca-bundle.crt \
  -n gravitee
```

### 2.4 Management API TLS Configuration

```yaml
# gravitee.yml - Management API TLS Configuration
jetty:
  secured: true
  ssl:
    keystore:
      type: PKCS12
      path: /etc/gravitee/certs/management-api-keystore.p12
      password: ${KEYSTORE_PASSWORD}
    truststore:
      type: PKCS12
      path: /etc/gravitee/certs/truststore.p12
      password: ${TRUSTSTORE_PASSWORD}
    clientAuth: false  # Set to true for mTLS
    
http:
  port: 8083
  host: 0.0.0.0
  
# Database TLS
management:
  mongodb:
    uri: mongodb://mongo-0.core.company.com:27017,mongo-1.core.company.com:27017,mongo-2.core.company.com:27017/gravitee?replicaSet=rs0&ssl=true
    sslEnabled: true
    trustStore:
      path: /etc/gravitee/certs/truststore.p12
      type: PKCS12
      password: ${TRUSTSTORE_PASSWORD}
      
# Elasticsearch TLS
analytics:
  elasticsearch:
    endpoints:
      - https://es-0.core.company.com:9200
      - https://es-1.core.company.com:9200
      - https://es-2.core.company.com:9200
    security:
      username: ${ES_USERNAME}
      password: ${ES_PASSWORD}
    ssl:
      keystore:
        type: PKCS12
        path: /etc/gravitee/certs/es-keystore.p12
        password: ${ES_KEYSTORE_PASSWORD}
```

### 2.5 API Gateway TLS Configuration

```yaml
# gravitee.yml - API Gateway TLS Configuration
http:
  port: 8082
  host: 0.0.0.0
  secured: true
  ssl:
    keystore:
      type: PKCS12
      path: /etc/gravitee/certs/gateway-keystore.p12
      password: ${KEYSTORE_PASSWORD}
    truststore:
      type: PKCS12
      path: /etc/gravitee/certs/truststore.p12
      password: ${TRUSTSTORE_PASSWORD}
    clientAuth: none  # Options: none, request, required (for mTLS)
    protocols:
      - TLSv1.2
      - TLSv1.3
    ciphers:
      - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
      - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
      - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
      - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256

# MongoDB TLS for config sync
management:
  mongodb:
    uri: mongodb://mongo-0.core.company.com:27017,mongo-1.core.company.com:27017,mongo-2.core.company.com:27017/gravitee?replicaSet=rs0&ssl=true
    sslEnabled: true
    trustStore:
      path: /etc/gravitee/certs/truststore.p12
      type: PKCS12
      password: ${TRUSTSTORE_PASSWORD}

# Redis TLS for rate limiting
ratelimit:
  type: redis
  redis:
    host: redis.core.company.com
    port: 6379
    ssl: true
    password: ${REDIS_PASSWORD}
    
# Elasticsearch TLS for analytics
reporters:
  elasticsearch:
    enabled: true
    endpoints:
      - https://es-0.core.company.com:9200
    security:
      username: ${ES_USERNAME}
      password: ${ES_PASSWORD}
    ssl:
      truststore:
        type: PKCS12
        path: /etc/gravitee/certs/truststore.p12
        password: ${TRUSTSTORE_PASSWORD}
```

### 2.6 Gateway mTLS Configuration for Backend APIs

```yaml
# gravitee.yml - mTLS for Backend Communication
http:
  ssl:
    # For outbound connections to backends
    trustAll: false
    truststore:
      type: PKCS12
      path: /etc/gravitee/certs/backend-truststore.p12
      password: ${BACKEND_TRUSTSTORE_PASSWORD}
    # Client certificate for mTLS to backends
    keystore:
      type: PKCS12
      path: /etc/gravitee/certs/gateway-client-keystore.p12
      password: ${GATEWAY_CLIENT_KEYSTORE_PASSWORD}
      
# Per-API mTLS configuration (via API definition)
# This is configured in the Console UI for each API endpoint
```

### 2.7 Console UI & Portal TLS (Ingress)

```yaml
# console-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gravitee-console-ingress
  namespace: gravitee
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - console.core.company.com
      secretName: console-tls
  rules:
    - host: console.core.company.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: gravitee-console-ui
                port:
                  number: 443

---
# portal-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gravitee-portal-ingress
  namespace: gravitee
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - portal.core.company.com
      secretName: portal-tls
  rules:
    - host: portal.core.company.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: gravitee-portal-ui
                port:
                  number: 443

---
# gateway-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gravitee-gateway-ingress
  namespace: gravitee
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - gateway.core.company.com
        - "*.gateway.core.company.com"
      secretName: gateway-tls
  rules:
    - host: gateway.core.company.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: gravitee-gateway
                port:
                  number: 8082
```

---

## 3. OAuth 2.0 / OIDC Configuration

### 3.1 OpenAM Client Registration

Register the following OAuth 2.0 clients in OpenAM:

| Client | Client ID | Grant Type | Redirect URIs |
|--------|-----------|------------|---------------|
| Console UI | gravitee-console | Authorization Code + PKCE | https://console.core.company.com/callback |
| Developer Portal | gravitee-portal | Authorization Code + PKCE | https://portal.core.company.com/callback |
| API Gateway | gravitee-gateway | Client Credentials | N/A |
| Management API | gravitee-management | Client Credentials | N/A |

### 3.2 OpenAM Client Configuration

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     OPENAM OAUTH 2.0 CLIENTS                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Client: gravitee-console                                         │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │ Client ID:        gravitee-console                               │   │
│  │ Client Secret:    <generated-secret>                             │   │
│  │ Grant Types:      authorization_code                             │   │
│  │ Response Types:   code                                           │   │
│  │ Token Auth:       client_secret_post                             │   │
│  │ Redirect URIs:    https://console.core.company.com/callback      │   │
│  │ Scopes:           openid profile email groups                    │   │
│  │ PKCE:             Required (S256)                                │   │
│  │ Token Lifetime:   Access: 3600s, Refresh: 86400s                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Client: gravitee-portal                                          │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │ Client ID:        gravitee-portal                                │   │
│  │ Client Secret:    <generated-secret>                             │   │
│  │ Grant Types:      authorization_code                             │   │
│  │ Response Types:   code                                           │   │
│  │ Token Auth:       client_secret_post                             │   │
│  │ Redirect URIs:    https://portal.core.company.com/callback       │   │
│  │ Scopes:           openid profile email                           │   │
│  │ PKCE:             Required (S256)                                │   │
│  │ Token Lifetime:   Access: 3600s, Refresh: 86400s                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Client: gravitee-gateway                                         │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │ Client ID:        gravitee-gateway                               │   │
│  │ Client Secret:    <generated-secret>                             │   │
│  │ Grant Types:      client_credentials                             │   │
│  │ Token Auth:       client_secret_basic                            │   │
│  │ Scopes:           introspect                                     │   │
│  │ Purpose:          Token introspection / JWKS fetch               │   │
│  │ Token Lifetime:   Access: 3600s                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Management API OAuth Configuration

```yaml
# gravitee.yml - Management API Identity Provider Configuration
security:
  providers:
    # OpenAM OIDC Provider
    - type: oidc
      id: openam
      clientId: gravitee-console
      clientSecret: ${OPENAM_CLIENT_SECRET}
      tokenIntrospectionEndpoint: https://sso.inner-dmz.company.com/oauth2/introspect
      tokenEndpoint: https://sso.inner-dmz.company.com/oauth2/access_token
      authorizeEndpoint: https://sso.inner-dmz.company.com/oauth2/authorize
      userInfoEndpoint: https://sso.inner-dmz.company.com/oauth2/userinfo
      jwksUri: https://sso.inner-dmz.company.com/oauth2/connect/jwk_uri
      
      # User info mapping
      userMapping:
        id: sub
        email: email
        firstname: given_name
        lastname: family_name
        picture: picture
      
      # Group mapping from OpenAM claims
      groupsMapping:
        - condition: "{#jsonPath(#profile, '$.groups').contains('gravitee-admins')}"
          groups:
            - "ADMIN"
        - condition: "{#jsonPath(#profile, '$.groups').contains('api-publishers')}"
          groups:
            - "API_PUBLISHER"
        - condition: "{#jsonPath(#profile, '$.groups').contains('api-consumers')}"
          groups:
            - "USER"
      
      # Role mapping
      roleMapping:
        - condition: "{#jsonPath(#profile, '$.groups').contains('gravitee-admins')}"
          roles:
            - "ORGANIZATION:ADMIN"
            - "ENVIRONMENT:ADMIN"
        - condition: "{#jsonPath(#profile, '$.groups').contains('api-publishers')}"
          roles:
            - "ENVIRONMENT:API_PUBLISHER"
            
      scopes:
        - openid
        - profile
        - email
        - groups
```

### 3.4 Console UI OAuth Configuration

```yaml
# constants.json - Console UI Configuration
{
  "baseURL": "https://api.core.company.com/management/",
  "portalURL": "https://portal.core.company.com/",
  
  "authentication": {
    "forceLogin": {
      "enabled": true
    },
    "localLogin": {
      "enabled": false
    }
  },
  
  "identityProviders": {
    "openam": {
      "enabled": true,
      "name": "Corporate SSO",
      "icon": "gio:openam",
      "clientId": "gravitee-console",
      "authorizationEndpoint": "https://sso.inner-dmz.company.com/oauth2/authorize",
      "tokenEndpoint": "https://sso.inner-dmz.company.com/oauth2/access_token",
      "userInfoEndpoint": "https://sso.inner-dmz.company.com/oauth2/userinfo",
      "logoutEndpoint": "https://sso.inner-dmz.company.com/oauth2/connect/endSession",
      "redirectUri": "https://console.core.company.com/callback",
      "scopes": ["openid", "profile", "email", "groups"],
      "pkce": true,
      "responseType": "code"
    }
  }
}
```

### 3.5 Developer Portal OAuth Configuration

```yaml
# constants.json - Portal Configuration
{
  "baseURL": "https://api.core.company.com/portal/",
  
  "authentication": {
    "forceLogin": {
      "enabled": false
    },
    "localLogin": {
      "enabled": false
    }
  },
  
  "identityProviders": {
    "openam": {
      "enabled": true,
      "name": "Corporate SSO",
      "icon": "gio:openam",
      "clientId": "gravitee-portal",
      "authorizationEndpoint": "https://sso.inner-dmz.company.com/oauth2/authorize",
      "tokenEndpoint": "https://sso.inner-dmz.company.com/oauth2/access_token",
      "userInfoEndpoint": "https://sso.inner-dmz.company.com/oauth2/userinfo",
      "logoutEndpoint": "https://sso.inner-dmz.company.com/oauth2/connect/endSession",
      "redirectUri": "https://portal.core.company.com/callback",
      "scopes": ["openid", "profile", "email"],
      "pkce": true,
      "responseType": "code"
    }
  }
}
```

### 3.6 API Gateway JWT Policy Configuration

```yaml
# gravitee.yml - Gateway JWT Resource Configuration
resources:
  # OpenAM JWKS Resource
  - name: openam-jwks
    type: oauth2-am-resource
    configuration:
      serverURL: https://sso.inner-dmz.company.com
      realm: /
      jwksUri: /oauth2/connect/jwk_uri
      introspectionEndpoint: /oauth2/introspect
      clientId: gravitee-gateway
      clientSecret: ${OPENAM_GATEWAY_CLIENT_SECRET}
      # Cache JWKS for performance
      cacheEnabled: true
      cacheTTL: 300
      
# JWT Validation Policy (applied per-API)
policies:
  jwt:
    signature: RS256
    publicKeyResolver: JWKS_URL
    jwksUrl: https://sso.inner-dmz.company.com/oauth2/connect/jwk_uri
    # Extract claims to request context
    extractClaims: true
    claimsToExtract:
      - sub
      - email
      - groups
      - scope
    # Propagate to backend
    propagateAuthHeader: true
```

### 3.7 API-Level OAuth Policy (Via Console UI)

```json
{
  "name": "OAuth2 JWT Validation",
  "policy": "jwt",
  "configuration": {
    "signature": "RS256",
    "publicKeyResolver": "JWKS_URL",
    "resolverParameter": "https://sso.inner-dmz.company.com/oauth2/connect/jwk_uri",
    "useSystemProxy": false,
    "extractClaims": true,
    "propagateAuthHeader": true,
    "clientIdClaim": "azp",
    "checkRequiredClaims": true,
    "requiredClaims": ["sub", "exp", "iat"],
    "validateExpDate": true,
    "validateNotBeforeDate": true
  }
}
```

---

## 4. Kubernetes Secrets for OAuth

```yaml
# oauth-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: gravitee-oauth-secrets
  namespace: gravitee
type: Opaque
stringData:
  # Console OAuth
  OPENAM_CONSOLE_CLIENT_ID: "gravitee-console"
  OPENAM_CONSOLE_CLIENT_SECRET: "<console-client-secret>"
  
  # Portal OAuth
  OPENAM_PORTAL_CLIENT_ID: "gravitee-portal"
  OPENAM_PORTAL_CLIENT_SECRET: "<portal-client-secret>"
  
  # Gateway OAuth (for token introspection)
  OPENAM_GATEWAY_CLIENT_ID: "gravitee-gateway"
  OPENAM_GATEWAY_CLIENT_SECRET: "<gateway-client-secret>"
  
  # Management API OAuth
  OPENAM_MGMT_CLIENT_ID: "gravitee-management"
  OPENAM_MGMT_CLIENT_SECRET: "<management-client-secret>"

---
# tls-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: gravitee-tls-secrets
  namespace: gravitee
type: Opaque
stringData:
  KEYSTORE_PASSWORD: "<keystore-password>"
  TRUSTSTORE_PASSWORD: "<truststore-password>"
  ES_KEYSTORE_PASSWORD: "<es-keystore-password>"
  BACKEND_TRUSTSTORE_PASSWORD: "<backend-truststore-password>"
  GATEWAY_CLIENT_KEYSTORE_PASSWORD: "<gateway-client-keystore-password>"
```

---

## 5. TLS Communication Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        TLS COMMUNICATION FLOWS                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  CORPORATE ZONE                                                              │
│  ┌─────────────┐                                                            │
│  │   Users     │                                                            │
│  │  (Browser)  │                                                            │
│  └──────┬──────┘                                                            │
│         │ TLS 1.2/1.3 (HTTPS :443)                                          │
│         │ Server Auth: Console/Portal Certificate                           │
│         ▼                                                                    │
│  ═══════════════════════════════════════════════════ FIREWALL 1 ════════   │
│         │                                                                    │
│  INNER DMZ                                                                   │
│         │                                                                    │
│         │  ┌─────────────────────────────────────────────────────┐         │
│         └──►        OpenAM SSO Server                             │         │
│            │  • HTTPS :443                                        │         │
│            │  • OAuth 2.0 / OIDC Endpoints                        │         │
│            │  • Server Certificate: sso.inner-dmz.company.com     │         │
│            └─────────────────────────────────────────────────────┘         │
│                                    ▲                                         │
│  ═══════════════════════════════════════════════════ FIREWALL 2 ════════   │
│                                    │ TLS 1.2/1.3                            │
│  CORE ZONE                         │                                         │
│                                    │                                         │
│  ┌───────────────────────────────────────────────────────────────────┐     │
│  │                    GRAVITEE APIM COMPONENTS                        │     │
│  │                                                                    │     │
│  │   ┌─────────────┐   TLS   ┌─────────────┐   TLS   ┌───────────┐  │     │
│  │   │ Console UI  │◄────────│ Mgmt API    │────────►│ MongoDB   │  │     │
│  │   │ :443 (HTTPS)│         │ :8083 (HTTPS)│         │ :27017    │  │     │
│  │   └─────────────┘         └──────┬──────┘         └───────────┘  │     │
│  │                                  │                                │     │
│  │   ┌─────────────┐   TLS         │ TLS            ┌───────────┐   │     │
│  │   │   Portal    │◄──────────────┤               │   Redis   │   │     │
│  │   │ :443 (HTTPS)│               │               │   :6379   │   │     │
│  │   └─────────────┘               │               └───────────┘   │     │
│  │                                 ▼                      ▲         │     │
│  │   ┌─────────────────────────────────────────┐         │ TLS     │     │
│  │   │         API Gateway Cluster              │─────────┘         │     │
│  │   │  :8082 (HTTPS) / :443 via Ingress        │                   │     │
│  │   │  • Server Certificate                    │    ┌───────────┐  │     │
│  │   │  • Client Certificate (mTLS to backends) │───►│   ES      │  │     │
│  │   └──────────────────────────────────────────┘    │   :9200   │  │     │
│  │                        │                          └───────────┘  │     │
│  └────────────────────────┼─────────────────────────────────────────┘     │
│                           │                                                 │
│  ═══════════════════════════════════════════════════ FIREWALL 3 ════════   │
│                           │ TLS 1.2/1.3 (mTLS Optional)                    │
│  BACKEND ZONE             ▼                                                 │
│  ┌───────────────────────────────────────────────────────────────────┐     │
│  │                   BACKEND APIs                                     │     │
│  │   • Microservices (HTTPS)                                         │     │
│  │   • Legacy Systems (HTTPS)                                        │     │
│  │   • Databases (TLS)                                               │     │
│  └───────────────────────────────────────────────────────────────────┘     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Troubleshooting TLS Issues

### 6.1 Common Issues and Solutions

| Issue | Symptoms | Solution |
|-------|----------|----------|
| Certificate not trusted | `PKIX path building failed` | Add CA certificate to truststore |
| Certificate expired | `NotAfter` error | Renew certificate and update secrets |
| Hostname mismatch | `No subject alternative names matching` | Ensure SAN includes all hostnames |
| Protocol mismatch | `No appropriate protocol` | Ensure TLSv1.2+ is enabled on both ends |
| Cipher suite mismatch | `No common cipher suites` | Configure matching cipher suites |

### 6.2 Testing TLS Connectivity

```bash
# Test TLS connection to OpenAM
openssl s_client -connect sso.inner-dmz.company.com:443 -servername sso.inner-dmz.company.com

# Test TLS connection to MongoDB
openssl s_client -connect mongo-0.core.company.com:27017 -servername mongo-0.core.company.com

# Test TLS connection to Elasticsearch
curl -v --cacert /path/to/ca.crt https://es-0.core.company.com:9200

# Verify certificate chain
openssl verify -CAfile /path/to/ca-bundle.crt /path/to/server.crt
```

---

## 7. Security Best Practices

1. **Use TLS 1.2 or higher** - Disable TLS 1.0 and 1.1
2. **Use strong cipher suites** - Prefer ECDHE for key exchange
3. **Rotate certificates before expiry** - Implement automated rotation
4. **Use separate certificates per service** - Don't share private keys
5. **Store secrets securely** - Use Kubernetes Secrets or HashiCorp Vault
6. **Enable HSTS** - Force HTTPS connections
7. **Validate certificates** - Don't set `trustAll: true` in production
8. **Use PKCE for public clients** - Console and Portal OAuth flows
9. **Limit token lifetimes** - Short-lived access tokens, reasonable refresh tokens
10. **Audit OAuth token usage** - Log token issuance and validation

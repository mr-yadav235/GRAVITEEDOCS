# Gravitee APIM Deployment Guide

## Overview

This guide provides step-by-step deployment instructions for Gravitee APIM components in a Kubernetes environment with OpenAM SSO integration.

---

## Architecture Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DEPLOYMENT ARCHITECTURE                             │
│                                                                              │
│   INNER DMZ                    CORE ZONE                                     │
│   ┌─────────────┐              ┌─────────────────────────────────────────┐  │
│   │   OpenAM    │◄────────────►│  Control Plane    │    Data Plane       │  │
│   │   (Pre-req) │              │  • Console UI     │    • US Gateway     │  │
│   └─────────────┘              │  • Portal         │    • EU Gateway     │  │
│                                │  • Mgmt API       │    • APAC Gateway   │  │
│                                │                   │                      │  │
│                                │  Data Services    │                      │  │
│                                │  • MongoDB        │                      │  │
│                                │  • Redis          │                      │  │
│                                │  • Elasticsearch  │                      │  │
│                                └─────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

### 1. Infrastructure Requirements

| Component | Minimum | Recommended | Purpose |
|-----------|---------|-------------|---------|
| **Kubernetes Cluster** | 1.21+ | 1.25+ | Container orchestration |
| **CPU** | 8 cores | 16 cores | All components |
| **Memory** | 16 GB | 32 GB | All components |
| **Storage** | 100 GB SSD | 500 GB SSD | Persistent data |
| **Nodes** | 3 | 6+ | HA deployment |

### 2. Software Requirements

| Software | Version | Purpose |
|----------|---------|---------|
| Helm | 3.10+ | Package management |
| kubectl | 1.21+ | Kubernetes CLI |
| OpenAM | 14.x+ | SSO Provider (pre-configured in Inner DMZ) |

### 3. Network Requirements

| Endpoint | Port | Access From |
|----------|------|-------------|
| OpenAM SSO | 443 | Core Zone |
| Kubernetes API | 6443 | Deployment machine |
| Container Registry | 443 | Kubernetes nodes |

### 4. DNS Records (Create Before Deployment)

```
console.internal.company.com     → Ingress Controller IP
portal.internal.company.com      → Ingress Controller IP
api-mgmt.internal.company.com    → Ingress Controller IP
api.internal.company.com         → Gateway Load Balancer IP
```

---

## Deployment Steps

### Step 1: Prepare Kubernetes Cluster

#### 1.1 Create Namespaces

```bash
# Create namespaces for Gravitee components
kubectl create namespace gravitee-control-plane
kubectl create namespace gravitee-data-plane
kubectl create namespace gravitee-data

# Label namespaces
kubectl label namespace gravitee-control-plane tier=control-plane zone=core
kubectl label namespace gravitee-data-plane tier=data-plane zone=core
kubectl label namespace gravitee-data tier=data zone=core

# Verify namespaces
kubectl get namespaces -l zone=core
```

#### 1.2 Create Storage Classes (if needed)

```yaml
# storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gravitee-ssd
provisioner: kubernetes.io/aws-ebs  # or your cloud provider
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

```bash
kubectl apply -f storage-class.yaml
```

---

### Step 2: Deploy Data Services

#### 2.1 Deploy MongoDB

```yaml
# mongodb-values.yaml
architecture: replicaset
replicaCount: 3

auth:
  enabled: true
  rootUser: root
  rootPassword: "${MONGODB_ROOT_PASSWORD}"
  databases:
    - gravitee
  usernames:
    - gravitee
  passwords:
    - "${MONGODB_GRAVITEE_PASSWORD}"

persistence:
  enabled: true
  storageClass: gravitee-ssd
  size: 50Gi

resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 2000m
    memory: 4Gi
```

```bash
# Add Bitnami Helm repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install MongoDB
helm install mongodb bitnami/mongodb \
  --namespace gravitee-data \
  --values mongodb-values.yaml \
  --set auth.rootPassword="${MONGODB_ROOT_PASSWORD}" \
  --set auth.passwords[0]="${MONGODB_GRAVITEE_PASSWORD}"

# Verify deployment
kubectl get pods -n gravitee-data -l app.kubernetes.io/name=mongodb
```

#### 2.2 Deploy Redis

```yaml
# redis-values.yaml
architecture: replication
auth:
  enabled: true
  password: "${REDIS_PASSWORD}"

replica:
  replicaCount: 3

master:
  persistence:
    enabled: true
    storageClass: gravitee-ssd
    size: 10Gi

resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 2Gi
```

```bash
# Install Redis
helm install redis bitnami/redis \
  --namespace gravitee-data \
  --values redis-values.yaml \
  --set auth.password="${REDIS_PASSWORD}"

# Verify deployment
kubectl get pods -n gravitee-data -l app.kubernetes.io/name=redis
```

#### 2.3 Deploy Elasticsearch

```yaml
# elasticsearch-values.yaml
replicas: 3
minimumMasterNodes: 2

resources:
  requests:
    cpu: 1000m
    memory: 2Gi
  limits:
    cpu: 2000m
    memory: 4Gi

volumeClaimTemplate:
  storageClassName: gravitee-ssd
  resources:
    requests:
      storage: 100Gi

esConfig:
  elasticsearch.yml: |
    cluster.name: "gravitee-es"
    network.host: 0.0.0.0
    xpack.security.enabled: true
```

```bash
# Add Elastic Helm repo
helm repo add elastic https://helm.elastic.co
helm repo update

# Install Elasticsearch
helm install elasticsearch elastic/elasticsearch \
  --namespace gravitee-data \
  --values elasticsearch-values.yaml

# Verify deployment
kubectl get pods -n gravitee-data -l app=elasticsearch-master
```

---

### Step 3: Create Secrets

#### 3.1 Create OpenAM SSO Secrets

```yaml
# openam-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: gravitee-openam-secrets
  namespace: gravitee-control-plane
type: Opaque
stringData:
  OPENAM_CLIENT_ID: "gravitee-console"
  OPENAM_CLIENT_SECRET: "${OPENAM_CLIENT_SECRET}"
  OPENAM_PORTAL_CLIENT_ID: "gravitee-portal"
  OPENAM_PORTAL_CLIENT_SECRET: "${OPENAM_PORTAL_CLIENT_SECRET}"
---
apiVersion: v1
kind: Secret
metadata:
  name: gravitee-openam-secrets
  namespace: gravitee-data-plane
type: Opaque
stringData:
  OPENAM_CLIENT_ID: "gravitee-gateway"
  OPENAM_CLIENT_SECRET: "${OPENAM_GATEWAY_CLIENT_SECRET}"
```

```bash
kubectl apply -f openam-secrets.yaml
```

#### 3.2 Create Database Secrets

```yaml
# database-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: gravitee-db-secrets
  namespace: gravitee-control-plane
type: Opaque
stringData:
  MONGODB_URI: "mongodb://gravitee:${MONGODB_GRAVITEE_PASSWORD}@mongodb-0.mongodb-headless.gravitee-data:27017,mongodb-1.mongodb-headless.gravitee-data:27017,mongodb-2.mongodb-headless.gravitee-data:27017/gravitee?replicaSet=rs0"
  REDIS_PASSWORD: "${REDIS_PASSWORD}"
  ELASTICSEARCH_PASSWORD: "${ES_PASSWORD}"
---
apiVersion: v1
kind: Secret
metadata:
  name: gravitee-db-secrets
  namespace: gravitee-data-plane
type: Opaque
stringData:
  MONGODB_URI: "mongodb://gravitee:${MONGODB_GRAVITEE_PASSWORD}@mongodb-0.mongodb-headless.gravitee-data:27017,mongodb-1.mongodb-headless.gravitee-data:27017,mongodb-2.mongodb-headless.gravitee-data:27017/gravitee?replicaSet=rs0"
  REDIS_PASSWORD: "${REDIS_PASSWORD}"
  ELASTICSEARCH_PASSWORD: "${ES_PASSWORD}"
```

```bash
kubectl apply -f database-secrets.yaml
```

---

### Step 4: Deploy Control Plane

#### 4.1 Create Control Plane Values

```yaml
# control-plane-values.yaml
global:
  kubeVersion: ""

# ============================================
# Management API
# ============================================
api:
  enabled: true
  name: gravitee-api
  replicaCount: 2
  
  image:
    repository: graviteeio/apim-management-api
    tag: 4.2.0
    pullPolicy: IfNotPresent
  
  env:
    - name: gravitee_security_providers_0_type
      value: "oidc"
    - name: gravitee_security_providers_0_id
      value: "openam"
    - name: gravitee_security_providers_0_clientId
      valueFrom:
        secretKeyRef:
          name: gravitee-openam-secrets
          key: OPENAM_CLIENT_ID
    - name: gravitee_security_providers_0_clientSecret
      valueFrom:
        secretKeyRef:
          name: gravitee-openam-secrets
          key: OPENAM_CLIENT_SECRET
    - name: gravitee_security_providers_0_tokenEndpoint
      value: "https://sso.inner-dmz.company.com/oauth2/token"
    - name: gravitee_security_providers_0_authorizeEndpoint
      value: "https://sso.inner-dmz.company.com/oauth2/authorize"
    - name: gravitee_security_providers_0_userInfoEndpoint
      value: "https://sso.inner-dmz.company.com/oauth2/userinfo"
  
  service:
    type: ClusterIP
    externalPort: 8083
    internalPort: 8083
  
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - host: api-mgmt.internal.company.com
        paths:
          - path: /
            pathType: Prefix
    tls:
      - secretName: gravitee-mgmt-api-tls
        hosts:
          - api-mgmt.internal.company.com
  
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 2Gi

# ============================================
# Console UI
# ============================================
ui:
  enabled: true
  name: gravitee-console
  replicaCount: 2
  
  image:
    repository: graviteeio/apim-management-ui
    tag: 4.2.0
    pullPolicy: IfNotPresent
  
  env:
    - name: MGMT_API_URL
      value: "https://api-mgmt.internal.company.com"
  
  service:
    type: ClusterIP
    externalPort: 8080
    internalPort: 8080
  
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - host: console.internal.company.com
        paths:
          - path: /
            pathType: Prefix
    tls:
      - secretName: gravitee-console-tls
        hosts:
          - console.internal.company.com
  
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 256Mi

# ============================================
# Developer Portal
# ============================================
portal:
  enabled: true
  name: gravitee-portal
  replicaCount: 2
  
  image:
    repository: graviteeio/apim-portal-ui
    tag: 4.2.0
    pullPolicy: IfNotPresent
  
  env:
    - name: PORTAL_API_URL
      value: "https://api-mgmt.internal.company.com"
  
  service:
    type: ClusterIP
    externalPort: 8080
    internalPort: 8080
  
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - host: portal.internal.company.com
        paths:
          - path: /
            pathType: Prefix
    tls:
      - secretName: gravitee-portal-tls
        hosts:
          - portal.internal.company.com
  
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 256Mi

# ============================================
# Disable Gateway in Control Plane
# ============================================
gateway:
  enabled: false

# ============================================
# Database Configuration
# ============================================
mongo:
  uri: mongodb://gravitee:password@mongodb-0.mongodb-headless.gravitee-data:27017,mongodb-1.mongodb-headless.gravitee-data:27017/gravitee?replicaSet=rs0

es:
  endpoints:
    - http://elasticsearch-master.gravitee-data:9200

redis:
  host: redis-master.gravitee-data
  port: 6379
```

#### 4.2 Deploy Control Plane

```bash
# Add Gravitee Helm repo
helm repo add graviteeio https://helm.gravitee.io
helm repo update

# Install Control Plane
helm install gravitee-control-plane graviteeio/apim \
  --namespace gravitee-control-plane \
  --values control-plane-values.yaml \
  --timeout 10m

# Verify deployment
kubectl get pods -n gravitee-control-plane
kubectl get ingress -n gravitee-control-plane
```

---

### Step 5: Deploy Data Plane (API Gateways)

#### 5.1 Create Data Plane Values

```yaml
# data-plane-values.yaml
global:
  kubeVersion: ""

# ============================================
# Disable Control Plane Components
# ============================================
api:
  enabled: false
ui:
  enabled: false
portal:
  enabled: false

# ============================================
# API Gateway
# ============================================
gateway:
  enabled: true
  name: gravitee-gateway
  replicaCount: 2
  
  image:
    repository: graviteeio/apim-gateway
    tag: 4.2.0
    pullPolicy: IfNotPresent
  
  # Sharding Tags for this cluster
  shardingTags: "internal-eu"
  
  env:
    # OpenAM JWT Validation
    - name: gravitee_policy_jwt_issuer_0_default
      value: "https://sso.inner-dmz.company.com/oauth2"
    - name: gravitee_policy_jwt_issuer_0_jwksUri
      value: "https://sso.inner-dmz.company.com/oauth2/jwks"
  
  service:
    type: ClusterIP
    externalPort: 8082
    internalPort: 8082
  
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - host: api.internal.company.com
        paths:
          - path: /
            pathType: Prefix
    tls:
      - secretName: gravitee-gateway-tls
        hosts:
          - api.internal.company.com
  
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70
    targetMemoryUtilizationPercentage: 80
  
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 2000m
      memory: 2Gi

# ============================================
# Database Configuration
# ============================================
mongo:
  uri: mongodb://gravitee:password@mongodb-0.mongodb-headless.gravitee-data:27017,mongodb-1.mongodb-headless.gravitee-data:27017/gravitee?replicaSet=rs0

es:
  endpoints:
    - http://elasticsearch-master.gravitee-data:9200

redis:
  host: redis-master.gravitee-data
  port: 6379
```

#### 5.2 Deploy Data Plane (Each Region)

```bash
# Deploy EU Gateway
helm install gravitee-gateway-eu graviteeio/apim \
  --namespace gravitee-data-plane \
  --values data-plane-values.yaml \
  --set gateway.shardingTags="internal-eu" \
  --timeout 10m

# Deploy US Gateway (in US K8s cluster)
helm install gravitee-gateway-us graviteeio/apim \
  --namespace gravitee-data-plane \
  --values data-plane-values.yaml \
  --set gateway.shardingTags="internal-us" \
  --timeout 10m

# Deploy APAC Gateway (in APAC K8s cluster)
helm install gravitee-gateway-apac graviteeio/apim \
  --namespace gravitee-data-plane \
  --values data-plane-values.yaml \
  --set gateway.shardingTags="internal-apac" \
  --timeout 10m

# Verify deployment
kubectl get pods -n gravitee-data-plane
kubectl get hpa -n gravitee-data-plane
```

---

### Step 6: Configure OpenAM Integration

#### 6.1 Register Gravitee Clients in OpenAM

Create the following OAuth 2.0 clients in OpenAM:

| Client ID | Redirect URIs | Grant Types |
|-----------|---------------|-------------|
| `gravitee-console` | `https://console.internal.company.com/callback` | authorization_code, refresh_token |
| `gravitee-portal` | `https://portal.internal.company.com/callback` | authorization_code, refresh_token |
| `gravitee-gateway` | N/A | client_credentials |

#### 6.2 Configure OpenAM Group Mapping

Create group mappings in OpenAM to Gravitee roles:

```json
{
  "groupsMapping": [
    {
      "condition": "{#jsonPath(#profile, '$.groups').contains('api-publishers')}",
      "groups": ["API_PUBLISHER"]
    },
    {
      "condition": "{#jsonPath(#profile, '$.groups').contains('platform-admins')}",
      "groups": ["ENVIRONMENT_ADMIN"]
    }
  ]
}
```

---

### Step 7: Apply Network Policies

```bash
# Apply network policies
kubectl apply -f network-policies/gateway-policy.yaml
kubectl apply -f network-policies/control-plane-policy.yaml

# Verify policies
kubectl get networkpolicies -A
```

---

### Step 8: Verify Deployment

#### 8.1 Check All Pods

```bash
# Check all Gravitee pods
kubectl get pods -n gravitee-control-plane
kubectl get pods -n gravitee-data-plane
kubectl get pods -n gravitee-data

# Expected output:
# NAME                                    READY   STATUS    RESTARTS   AGE
# gravitee-api-xxxxxxxxxx-xxxxx           1/1     Running   0          5m
# gravitee-console-xxxxxxxxxx-xxxxx       1/1     Running   0          5m
# gravitee-portal-xxxxxxxxxx-xxxxx        1/1     Running   0          5m
# gravitee-gateway-xxxxxxxxxx-xxxxx       1/1     Running   0          5m
```

#### 8.2 Check Services and Ingresses

```bash
# Check services
kubectl get svc -n gravitee-control-plane
kubectl get svc -n gravitee-data-plane

# Check ingresses
kubectl get ingress -A

# Expected URLs:
# - https://console.internal.company.com
# - https://portal.internal.company.com
# - https://api-mgmt.internal.company.com
# - https://api.internal.company.com
```

#### 8.3 Health Checks

```bash
# Check Management API health
curl -k https://api-mgmt.internal.company.com/management/organizations/DEFAULT/environments/DEFAULT/apis

# Check Gateway health
curl -k https://api.internal.company.com/_node/health

# Expected response: {"status": "UP"}
```

#### 8.4 Test Authentication Flow

```bash
# 1. Access Console UI
# Browser: https://console.internal.company.com
# Expected: Redirect to OpenAM login page

# 2. After login, verify JWT token
# Check browser cookies/localStorage for gravitee session

# 3. Test API call with OAuth token
TOKEN=$(curl -X POST https://sso.inner-dmz.company.com/oauth2/token \
  -d "grant_type=client_credentials" \
  -d "client_id=my-app" \
  -d "client_secret=xxx" \
  -d "scope=api:read" | jq -r '.access_token')

curl -H "Authorization: Bearer $TOKEN" \
  https://api.internal.company.com/my-api/v1/health
```

---

## Deployment Checklist

### Pre-Deployment

- [ ] Kubernetes cluster provisioned
- [ ] Namespaces created
- [ ] Storage classes configured
- [ ] DNS records created
- [ ] TLS certificates obtained
- [ ] OpenAM clients registered
- [ ] Firewall rules applied

### Data Services

- [ ] MongoDB deployed (3 replicas)
- [ ] Redis deployed (3 replicas)
- [ ] Elasticsearch deployed (3 replicas)
- [ ] Secrets created

### Control Plane

- [ ] Management API deployed (2 replicas)
- [ ] Console UI deployed (2 replicas)
- [ ] Developer Portal deployed (2 replicas)
- [ ] Ingress configured
- [ ] OpenAM integration tested

### Data Plane

- [ ] US Gateway deployed (2+ replicas)
- [ ] EU Gateway deployed (2+ replicas)
- [ ] APAC Gateway deployed (2+ replicas)
- [ ] HPA configured
- [ ] Sharding tags assigned

### Post-Deployment

- [ ] Health checks passing
- [ ] Login flow working
- [ ] API calls authenticated
- [ ] Analytics flowing to ES
- [ ] Rate limiting working

---

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Gateway can't reach MongoDB | Network policy blocking | Check NetworkPolicy egress rules |
| Login redirects fail | OpenAM client misconfigured | Verify redirect URIs in OpenAM |
| Token validation fails | JWKS endpoint unreachable | Check firewall to Inner DMZ |
| Console shows blank | Management API unreachable | Check Ingress and service |

### Useful Commands

```bash
# View logs
kubectl logs -n gravitee-control-plane -l app=gravitee-api -f
kubectl logs -n gravitee-data-plane -l app=gravitee-gateway -f

# Describe pod issues
kubectl describe pod -n gravitee-control-plane <pod-name>

# Check events
kubectl get events -n gravitee-control-plane --sort-by='.lastTimestamp'

# Test internal connectivity
kubectl run test-pod --rm -it --image=curlimages/curl -- sh
curl http://gravitee-api.gravitee-control-plane:8083/_node/health
```

---

## Related Documents

| Document | Description |
|----------|-------------|
| [Firewall Requirements](./FIREWALL-REQUIREMENTS.md) | Network security rules |
| [Architecture Overview](./HLD-CORE-ZONE-ARCHITECTURE.md) | System architecture |
| [User Flows](./USER-INTERACTION-FLOWS.md) | Authentication flows |
| [API Group RBAC](./API-GROUP-RBAC.md) | Permission model |

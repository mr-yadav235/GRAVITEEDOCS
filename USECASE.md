# Use Case: LDAP Authentication with Gravitee Authorization

## Document Purpose

This document describes the integration requirements between **Corporate LDAP/OpenAM** and **Gravitee API Management Platform** for the IAM team review.

**Key Principle**: 
- **LDAP/OpenAM** handles **Authentication** (verifies user identity)
- **Gravitee** handles **Authorization** (controls API access via internal Groups)

---

## 1. What We Want to Achieve

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │                    AUTHENTICATION (IAM Team Responsibility)                  │   │
│   │                    "Who are you?"                                            │   │
│   │                                                                              │   │
│   │   User Login ──────► OpenAM SSO ──────► LDAP                                │   │
│   │   (Credentials)      (OAuth/OIDC)       (Verify Identity)                   │   │
│   │                                                                              │   │
│   │   Returns: JWT Token with user identity                                      │   │
│   │   - Username (sub)                                                           │   │
│   │   - Email                                                                    │   │
│   │   - Display Name                                                             │   │
│   │   - NO group/role claims needed                                              │   │
│   │                                                                              │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                   │                                                  │
│                                   │ User Identity Only                               │
│                                   ▼                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │                    AUTHORIZATION (Gravitee Responsibility)                   │   │
│   │                    "What can you access?"                                    │   │
│   │                                                                              │   │
│   │   Gravitee receives authenticated user identity                              │   │
│   │   Gravitee manages:                                                          │   │
│   │   - Groups (Wealth Management, Core Tools, etc.)                            │   │
│   │   - Roles (API_USER, API_OWNER)                                             │   │
│   │   - API Access Permissions                                                   │   │
│   │   - Application Subscriptions                                                │   │
│   │                                                                              │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. What We Need from IAM Team

### 2.1 OpenAM OAuth2/OIDC Configuration

| Requirement | Description |
|-------------|-------------|
| **OAuth2 Client Registration** | Register Gravitee Console and Portal as OAuth2 clients |
| **Redirect URIs** | Configure allowed callback URLs for Gravitee applications |
| **Scopes** | Enable `openid`, `profile`, `email` scopes |
| **Token Format** | JWT tokens with standard claims |

### 2.2 Required JWT Claims

| Claim | Required | Description |
|-------|----------|-------------|
| `sub` | Yes | Unique user identifier (username) |
| `email` | Yes | User email address |
| `name` | Yes | Display name |
| `given_name` | Optional | First name |
| `family_name` | Optional | Last name |
| `groups` | **NOT Required** | Gravitee will manage groups internally |

### 2.3 Endpoints Required

| Endpoint | Purpose |
|----------|---------|
| Authorization Endpoint | User login redirect |
| Token Endpoint | Exchange code for tokens |
| UserInfo Endpoint | Retrieve user profile |
| JWKS Endpoint | Validate JWT signatures |

---

## 3. User Onboarding Flow (IAC Process)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         USER ONBOARDING VIA IAC PROCESS                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   STEP 1: User Exists in LDAP (Prerequisite)                                        │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │ User must already exist in corporate LDAP                                     │ │
│   │ (Managed by HR/IT through existing employee onboarding process)               │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│   STEP 2: User Raises IAC Request                                                   │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │ User submits IAC request specifying:                                          │ │
│   │ - Requested Gravitee Group (e.g., "Wealth Management")                        │ │
│   │ - Business Justification                                                      │ │
│   │ - Manager Approval                                                            │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                          │                                                          │
│                          ▼                                                          │
│   STEP 3: Approval Workflow                                                         │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │ IAC Request reviewed by:                                                      │ │
│   │ - User's Manager                                                              │ │
│   │ - Group Owner/Team Lead                                                       │ │
│   │ - Platform Admin (if required)                                                │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                          │                                                          │
│                          ▼                                                          │
│   STEP 4: Platform Admin Adds User to Gravitee Group                               │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │ Upon IAC approval, Platform Admin:                                            │ │
│   │ - Adds user to requested Gravitee Group                                       │ │
│   │ - Assigns appropriate role (API_USER or API_OWNER)                            │ │
│   │ - User can now access APIs within that group                                  │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Gravitee Groups Structure

### 4.1 Business Groups

| Group Name | Description | Example APIs |
|------------|-------------|--------------|
| **Wealth Management** | Wealth management team | Portfolio API, Investment API, Advisory API |
| **Core Tools** | Platform/DevOps team | Monitoring API, Logging API, CI/CD API |
| **Payments** | Payment processing team | Transfers API, Billing API, Settlements API |
| **Risk & Compliance** | Risk management team | Risk Assessment API, Compliance API |
| **Customer Services** | Customer-facing team | CRM API, Support API, Notifications API |

### 4.2 Visual Structure

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           GRAVITEE GROUPS                                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   ┌───────────────────────────────────────────────────────────────────────────────┐ │
│   │  WEALTH MANAGEMENT GROUP                                                       │ │
│   │                                                                                │ │
│   │  Members:                          APIs:                                       │ │
│   │  - john.smith (API_OWNER)          - Portfolio API                            │ │
│   │  - jane.doe (API_OWNER)            - Investment API                           │ │
│   │  - emma.dev (API_USER)             - Advisory API                             │ │
│   │                                    - Asset Management API                      │ │
│   │  Applications:                                                                 │ │
│   │  - WM-Mobile-App                                                              │ │
│   │  - WM-Web-Portal                                                              │ │
│   └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│   ┌───────────────────────────────────────────────────────────────────────────────┐ │
│   │  CORE TOOLS GROUP                                                              │ │
│   │                                                                                │ │
│   │  Members:                          APIs:                                       │ │
│   │  - mike.ops (API_OWNER)            - DevOps API                               │ │
│   │  - sarah.admin (API_OWNER)         - Monitoring API                           │ │
│   │  - tom.dev (API_USER)              - Logging API                              │ │
│   │                                    - CI/CD API                                 │ │
│   │  Applications:                                                                 │ │
│   │  - Internal-Dashboard                                                          │ │
│   │  - Ops-Automation                                                              │ │
│   └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Authentication Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         USER LOGIN FLOW                                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐      ┌────────────┐ │
│   │    User      │      │  Gravitee    │      │   OpenAM     │      │   LDAP     │ │
│   │              │      │  Console     │      │   SSO        │      │            │ │
│   └──────┬───────┘      └──────┬───────┘      └──────┬───────┘      └─────┬──────┘ │
│          │                     │                     │                    │        │
│          │  1. Access Console  │                     │                    │        │
│          │────────────────────►│                     │                    │        │
│          │                     │                     │                    │        │
│          │                     │  2. Redirect to SSO │                    │        │
│          │◄────────────────────│────────────────────►│                    │        │
│          │                     │                     │                    │        │
│          │  3. Enter Credentials                     │                    │        │
│          │──────────────────────────────────────────►│                    │        │
│          │                     │                     │                    │        │
│          │                     │                     │  4. Verify User    │        │
│          │                     │                     │───────────────────►│        │
│          │                     │                     │                    │        │
│          │                     │                     │  5. User Valid ✓   │        │
│          │                     │                     │◄───────────────────│        │
│          │                     │                     │                    │        │
│          │  6. JWT Token (identity only)             │                    │        │
│          │◄──────────────────────────────────────────│                    │        │
│          │                     │                     │                    │        │
│          │  7. Access with Token                     │                    │        │
│          │────────────────────►│                     │                    │        │
│          │                     │                     │                    │        │
│          │                     │  8. Gravitee checks │                    │        │
│          │                     │     internal groups │                    │        │
│          │                     │     & permissions   │                    │        │
│          │                     │                     │                    │        │
│          │  9. Show user's APIs│                     │                    │        │
│          │◄────────────────────│                     │                    │        │
│          │                     │                     │                    │        │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Roles and Permissions (Managed in Gravitee)

| Role | Description |
|------|-------------|
| **API_USER** | Can view and subscribe to APIs within their group |
| **API_OWNER** | Can create, edit, and manage APIs within their group |
| **PLATFORM_ADMIN** | Can manage all groups, users, and APIs across platform |

---

## 7. Key Benefits

| Benefit | Description |
|---------|-------------|
| **Separation of Concerns** | IAM team manages identity; API team manages API access |
| **No LDAP Group Dependency** | No need to create/manage API-specific groups in LDAP |
| **Self-Service** | Platform Admin can manage group memberships without IAM tickets |
| **Flexibility** | API access changes don't require LDAP modifications |
| **Audit Trail** | All permission changes tracked within Gravitee |
| **IAC Compliance** | Access follows standard IAC approval workflow |

---

## 8. Summary of Responsibilities

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        RESPONSIBILITY MATRIX                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   ┌─────────────────────────────────┐   ┌─────────────────────────────────────────┐ │
│   │       IAM TEAM                  │   │       GRAVITEE PLATFORM TEAM            │ │
│   ├─────────────────────────────────┤   ├─────────────────────────────────────────┤ │
│   │                                 │   │                                         │ │
│   │  ✓ User identity in LDAP       │   │  ✓ Create/manage Gravitee Groups       │ │
│   │  ✓ OpenAM OAuth2 client setup  │   │  ✓ Assign users to Groups (post IAC)   │ │
│   │  ✓ SSO authentication          │   │  ✓ Define roles and permissions        │ │
│   │  ✓ JWT token issuance          │   │  ✓ Manage API visibility               │ │
│   │  ✓ Password policies           │   │  ✓ Manage application subscriptions    │ │
│   │  ✓ MFA (if required)           │   │  ✓ API access audit and reporting      │ │
│   │                                 │   │                                         │ │
│   │  ✗ API-specific groups         │   │                                         │ │
│   │  ✗ API access management       │   │                                         │ │
│   │                                 │   │                                         │ │
│   └─────────────────────────────────┘   └─────────────────────────────────────────┘ │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Action Items for IAM Team

| # | Action Item | Details |
|---|-------------|---------|
| 1 | Register OAuth2 Client | Create OAuth2 client for Gravitee Console |
| 2 | Register OAuth2 Client | Create OAuth2 client for Gravitee Developer Portal |
| 3 | Configure Redirect URIs | Whitelist Gravitee callback URLs |
| 4 | Enable Required Scopes | `openid`, `profile`, `email` |
| 5 | Provide Endpoints | Share authorization, token, userinfo, JWKS URLs |
| 6 | Share Client Credentials | Provide client_id and client_secret for registered clients |

---

## 10. Related Documents

| Document | Description |
|----------|-------------|
| [JSZ-ARCHITECTURE.md](./JSZ-ARCHITECTURE.md) | Overall architecture |
| [INTERNAL-GATEWAY-ARCHITECTURE.md](./INTERNAL-GATEWAY-ARCHITECTURE.md) | Gateway architecture |

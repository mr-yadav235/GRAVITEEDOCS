# API Lifecycle & Governance Process

## Document Purpose

This document defines the **API lifecycle management process** and **governance framework** to ensure proper approval and oversight before any API or application is onboarded to the Gravitee API Management Platform.

---

## 1. Access Model

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              ACCESS MODEL                                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │                     DEVELOPER PORTAL ACCESS                                  │   │
│   │                                                                              │   │
│   │   ✓ OPEN TO ALL authenticated corporate users                               │   │
│   │   ✓ No approval required for portal access                                   │   │
│   │   ✓ Users can browse API catalog                                            │   │
│   │   ✓ Users can view API documentation                                        │   │
│   │   ✓ Users can create applications                                           │   │
│   │                                                                              │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │                     API SUBSCRIPTION                                         │   │
│   │                                                                              │   │
│   │   ✗ Requires approval                                                        │   │
│   │   ✗ Approval matrix based on API classification                             │   │
│   │   ✗ API Owner reviews and approves subscription requests                    │   │
│   │                                                                              │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Governance Principles

| Principle | Description |
|-----------|-------------|
| **Open Portal Access** | Developer Portal is open to all authenticated users |
| **Subscription Approval Required** | API access requires subscription approval from API Owner |
| **Application-First Approach** | Applications must be registered before their APIs are onboarded |
| **Multi-Level Approval** | All APIs require business, technical, and security approvals |
| **Lifecycle Tracking** | Every API state change is audited and tracked |

---

## 3. API Lifecycle Stages

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           API LIFECYCLE STAGES                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    │
│    │          │    │          │    │          │    │          │    │          │    │
│    │ PROPOSED │───►│ APPROVED │───►│   UAT    │───►│   LIVE   │───►│ RETIRED  │    │
│    │          │    │          │    │          │    │          │    │          │    │
│    └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘    │
│         │              │               │               │               │            │
│         │              │               │               │               │            │
│         ▼              ▼               ▼               ▼               ▼            │
│    ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐       │
│    │ Business │   │ Technical│   │ Security │   │ Ongoing  │   │ Sunset   │       │
│    │ Review   │   │ Review   │   │ Review   │   │ Monitor  │   │ Plan     │       │
│    └──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘       │
│                                                                                      │
│    Can be REJECTED at any stage and returned to PROPOSED with feedback              │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Application Onboarding (Pre-requisite for APIs)

Before any API can be created, the owning **Application** must be registered and approved.

### 4.1 Application Registration Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    APPLICATION ONBOARDING PROCESS                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   STEP 1: Application Registration Request                                          │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │ Requestor submits application registration form:                              │ │
│   │ - Application Name                                                            │ │
│   │ - Business Owner (Product Owner / Manager)                                    │ │
│   │ - Technical Owner (Tech Lead / Architect)                                     │ │
│   │ - Business Unit / Cost Center                                                 │ │
│   │ - Application Description & Purpose                                           │ │
│   │ - Data Classification (Public / Internal / Confidential / Restricted)         │ │
│   │ - Expected API Count                                                          │ │
│   │ - Target Consumers (Internal / External / Both)                               │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                          │                                                          │
│                          ▼                                                          │
│   STEP 2: Business Approval                                                         │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │ Approved by: Business Unit Head / Director                                    │ │
│   │ Validates:                                                                    │ │
│   │ - Business justification                                                      │ │
│   │ - Budget allocation                                                           │ │
│   │ - Resource availability                                                       │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                          │                                                          │
│                          ▼                                                          │
│   STEP 3: Architecture Review                                                       │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │ Approved by: Enterprise Architecture Team                                     │ │
│   │ Validates:                                                                    │ │
│   │ - Alignment with enterprise standards                                         │ │
│   │ - No duplication with existing applications                                   │ │
│   │ - Integration patterns                                                        │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                          │                                                          │
│                          ▼                                                          │
│   STEP 4: Platform Admin Creates Application                                        │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │ Platform Admin:                                                               │ │
│   │ - Creates Application in Gravitee                                             │ │
│   │ - Assigns to appropriate Gravitee Group                                       │ │
│   │ - Assigns Business Owner & Technical Owner                                    │ │
│   │ - Application is now ready to host APIs                                       │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Application Registration Requirements

| Field | Required | Description |
|-------|----------|-------------|
| Application Name | Yes | Unique identifier following naming convention |
| Business Owner | Yes | Accountable for business outcomes |
| Technical Owner | Yes | Accountable for technical delivery |
| Business Unit | Yes | Cost center for chargebacks |
| Data Classification | Yes | Determines security requirements |
| Description | Yes | Purpose and scope of the application |

---

## 5. API Onboarding Process

Once an Application is approved, APIs can be proposed for that application.

### 5.1 API Proposal & Approval Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        API ONBOARDING APPROVAL FLOW                                  │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │  STAGE 1: API PROPOSAL                                        [PROPOSED]    │   │
│   ├─────────────────────────────────────────────────────────────────────────────┤   │
│   │                                                                             │   │
│   │  Requestor: API Developer / Technical Owner                                 │   │
│   │                                                                             │   │
│   │  Required Information:                                                      │   │
│   │  ┌─────────────────────────────────────────────────────────────────────┐   │   │
│   │  │ • API Name & Version                                                │   │   │
│   │  │ • Parent Application (must be approved)                             │   │   │
│   │  │ • API Specification (OpenAPI/Swagger)                               │   │   │
│   │  │ • Backend Service Details                                           │   │   │
│   │  │ • Expected Traffic Volume                                           │   │   │
│   │  │ • Target Consumers                                                  │   │   │
│   │  │ • Data Elements (especially PII/MNPI)                               │   │   │
│   │  │ • Rate Limiting Requirements                                        │   │   │
│   │  │ • Authentication Method (OAuth, API Key, etc.)                      │   │   │
│   │  └─────────────────────────────────────────────────────────────────────┘   │   │
│   │                                                                             │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                            │
│                                        ▼                                            │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │  STAGE 2: TECHNICAL REVIEW                                    [IN REVIEW]   │   │
│   ├─────────────────────────────────────────────────────────────────────────────┤   │
│   │                                                                             │   │
│   │  Reviewer: API Platform Team / Tech Lead                                    │   │
│   │                                                                             │   │
│   │  Review Checklist:                                                          │   │
│   │  ┌─────────────────────────────────────────────────────────────────────┐   │   │
│   │  │ □ API spec follows design standards                                 │   │   │
│   │  │ □ Naming conventions followed                                       │   │   │
│   │  │ □ Versioning strategy aligned                                       │   │   │
│   │  │ □ Error handling standards met                                      │   │   │
│   │  │ □ Backend service is accessible                                     │   │   │
│   │  │ □ No duplicate API exists                                           │   │   │
│   │  │ □ Rate limits are appropriate                                       │   │   │
│   │  └─────────────────────────────────────────────────────────────────────┘   │   │
│   │                                                                             │   │
│   │  Outcome: APPROVE / REJECT with feedback                                    │   │
│   │                                                                             │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                            │
│                                        ▼                                            │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │  STAGE 3: SECURITY REVIEW                                     [IN REVIEW]   │   │
│   ├─────────────────────────────────────────────────────────────────────────────┤   │
│   │                                                                             │   │
│   │  Reviewer: Information Security Team                                        │   │
│   │                                                                             │   │
│   │  Review Checklist:                                                          │   │
│   │  ┌─────────────────────────────────────────────────────────────────────┐   │   │
│   │  │ □ Authentication method appropriate for data classification         │   │   │
│   │  │ □ PII/MNPI data handling compliant                                  │   │   │
│   │  │ □ Data masking configured where required                            │   │   │
│   │  │ □ TLS requirements met                                              │   │   │
│   │  │ □ Logging excludes sensitive data                                   │   │   │
│   │  │ □ Rate limiting prevents abuse                                      │   │   │
│   │  │ □ IP restrictions (if required)                                     │   │   │
│   │  └─────────────────────────────────────────────────────────────────────┘   │   │
│   │                                                                             │   │
│   │  Outcome: APPROVE / REJECT with feedback                                    │   │
│   │                                                                             │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                            │
│                                        ▼                                            │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │  STAGE 4: APPROVED - DEPLOY TO UAT                            [APPROVED]    │   │
│   ├─────────────────────────────────────────────────────────────────────────────┤   │
│   │                                                                             │   │
│   │  Action by: Platform Admin                                                  │   │
│   │                                                                             │   │
│   │  • API deployed to UAT environment                                          │   │
│   │  • API marked as "UAT" state in Gravitee                                    │   │
│   │  • Consumer testing can begin                                               │   │
│   │                                                                             │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                            │
│                                        ▼                                            │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │  STAGE 5: UAT SIGN-OFF                                        [UAT]         │   │
│   ├─────────────────────────────────────────────────────────────────────────────┤   │
│   │                                                                             │   │
│   │  Approved by: Business Owner + Technical Owner                              │   │
│   │                                                                             │   │
│   │  • Functional testing completed                                             │   │
│   │  • Performance testing completed                                            │   │
│   │  • Consumer integration verified                                            │   │
│   │  • Sign-off documented                                                      │   │
│   │                                                                             │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                            │
│                                        ▼                                            │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │  STAGE 6: PRODUCTION DEPLOYMENT                               [LIVE]        │   │
│   ├─────────────────────────────────────────────────────────────────────────────┤   │
│   │                                                                             │   │
│   │  Action by: Platform Admin (following Change Management process)            │   │
│   │                                                                             │   │
│   │  • API deployed to Production                                               │   │
│   │  • API state changed to "PUBLISHED"                                         │   │
│   │  • API visible in Developer Portal                                          │   │
│   │  • Consumers can request subscriptions                                      │   │
│   │  • Monitoring enabled                                                       │   │
│   │                                                                             │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. API Subscription Approval Matrix

### 6.1 Consumer Subscription Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    API SUBSCRIPTION APPROVAL FLOW                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │  Any authenticated user can access Developer Portal                          │   │
│   │  → Browse API catalog                                                        │   │
│   │  → View API documentation                                                    │   │
│   │  → Create their own application                                              │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                            │
│                                        ▼                                            │
│   ┌──────────────┐                                                                  │
│   │   Consumer   │  1. Clicks "Subscribe" on desired API                           │
│   │              │                                                                  │
│   └──────┬───────┘                                                                  │
│          │                                                                          │
│          ▼                                                                          │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │ Subscription Request includes:                                                │ │
│   │ - Consumer Application                                                        │ │
│   │ - Target API                                                                  │ │
│   │ - Subscription Plan (Bronze / Silver / Gold)                                  │ │
│   │ - Business Justification (free text)                                          │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│          │                                                                          │
│          ▼                                                                          │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │                     APPROVAL BASED ON API CLASSIFICATION                      │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│          │                                                                          │
│          ▼                                                                          │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │ If APPROVED:                                                                  │ │
│   │ - Subscription activated                                                      │ │
│   │ - API Key / OAuth credentials issued                                          │ │
│   │ - Consumer can access API immediately                                         │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Subscription Approval Matrix

| API Classification | Approval Required | Approver | Auto-Approve Option |
|--------------------|-------------------|----------|---------------------|
| **Public** | No | - | Yes (auto-approve enabled) |
| **Internal** | Yes | API Owner | No |
| **Confidential** | Yes | API Owner + Business Owner | No |
| **Restricted** | Yes | API Owner + Business Owner + Security Team | No |

### 6.3 Subscription Plans

| Plan | Rate Limit | Use Case | Approval |
|------|------------|----------|----------|
| **Bronze** | 100 req/min | Development / Testing | Standard |
| **Silver** | 1,000 req/min | Low-volume production | Standard |
| **Gold** | 10,000 req/min | High-volume production | API Owner + Capacity Review |
| **Unlimited** | No limit | Critical systems | API Owner + Business Owner |

---

## 7. Approval Roles & Responsibilities

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        APPROVAL ROLES MATRIX                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   ┌─────────────────────┬──────────────────────────────────────────────────────┐   │
│   │ ROLE                │ RESPONSIBILITIES                                     │   │
│   ├─────────────────────┼──────────────────────────────────────────────────────┤   │
│   │                     │                                                      │   │
│   │ Business Owner      │ • Approves business justification                    │   │
│   │                     │ • Owns API from business perspective                 │   │
│   │                     │ • Signs off UAT                                      │   │
│   │                     │ • Approves high-tier subscriptions                   │   │
│   │                     │                                                      │   │
│   ├─────────────────────┼──────────────────────────────────────────────────────┤   │
│   │                     │                                                      │   │
│   │ Technical Owner     │ • Defines API specification                          │   │
│   │                     │ • Owns API from technical perspective                │   │
│   │                     │ • Signs off UAT                                      │   │
│   │                     │ • Responsible for backend service                    │   │
│   │                     │                                                      │   │
│   ├─────────────────────┼──────────────────────────────────────────────────────┤   │
│   │                     │                                                      │   │
│   │ API Owner           │ • Reviews and approves subscription requests         │   │
│   │ (in Gravitee)       │ • Manages API visibility                             │   │
│   │                     │ • Can delegate to group members                      │   │
│   │                     │                                                      │   │
│   ├─────────────────────┼──────────────────────────────────────────────────────┤   │
│   │                     │                                                      │   │
│   │ API Platform Team   │ • Technical review of API design                     │   │
│   │                     │ • Validates standards compliance                     │   │
│   │                     │ • Approves/rejects technical aspects                 │   │
│   │                     │                                                      │   │
│   ├─────────────────────┼──────────────────────────────────────────────────────┤   │
│   │                     │                                                      │   │
│   │ Security Team       │ • Security review for API onboarding                 │   │
│   │                     │ • Data classification validation                     │   │
│   │                     │ • Approves Restricted API subscriptions              │   │
│   │                     │                                                      │   │
│   ├─────────────────────┼──────────────────────────────────────────────────────┤   │
│   │                     │                                                      │   │
│   │ Platform Admin      │ • Creates applications/APIs in Gravitee             │   │
│   │                     │ • Deploys to UAT/Production                          │   │
│   │                     │ • No approval authority (executor only)              │   │
│   │                     │                                                      │   │
│   └─────────────────────┴──────────────────────────────────────────────────────┘   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. API Deprecation & Retirement

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                       API RETIREMENT PROCESS                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   STEP 1: Deprecation Notice                                                        │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │ • API marked as DEPRECATED in Gravitee                                        │ │
│   │ • All active subscribers notified                                             │ │
│   │ • Deprecation timeline communicated (minimum 90 days)                         │ │
│   │ • New subscriptions blocked                                                   │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                          │                                                          │
│                          ▼                                                          │
│   STEP 2: Migration Period                                                          │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │ • Consumers migrate to replacement API (if available)                         │ │
│   │ • Support provided for migration                                              │ │
│   │ • Regular reminders sent to active consumers                                  │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                          │                                                          │
│                          ▼                                                          │
│   STEP 3: Final Approval for Retirement                                            │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │ • Business Owner confirms retirement                                          │ │
│   │ • Active subscribers count reviewed                                           │ │
│   │ • Exception handling for remaining consumers                                  │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                          │                                                          │
│                          ▼                                                          │
│   STEP 4: Retirement                                                                │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │ • API unpublished from Gravitee                                               │ │
│   │ • All subscriptions terminated                                                │ │
│   │ • API archived (not deleted for audit)                                        │ │
│   │ • Documentation retained for compliance                                       │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Governance Controls Summary

| Control | Purpose | Enforced By |
|---------|---------|-------------|
| **Open Portal Access** | Browse & discover APIs without barriers | Platform Policy |
| **Application Pre-registration** | No APIs without approved application | Platform Admin |
| **Multi-stage API Approval** | Technical + Security review for new APIs | Workflow System |
| **Subscription Approval Matrix** | Classification-based approval for access | Gravitee Platform |
| **Audit Trail** | All actions logged | Gravitee Analytics |
| **Deprecation Policy** | Minimum 90-day notice | Process Policy |

---

## 10. API States in Gravitee

| State | Description | Visible in Portal |
|-------|-------------|-------------------|
| **DRAFT** | API proposed, awaiting approval | No |
| **IN_REVIEW** | Under technical/security review | No |
| **UAT** | Approved, deployed to UAT environment | No |
| **PUBLISHED** | Live in Production | Yes - consumers can subscribe |
| **DEPRECATED** | Marked for retirement | Yes - with deprecation warning |
| **RETIRED** | No longer accessible | No |

---

## 11. Summary: End-to-End Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        END-TO-END GOVERNANCE FLOW                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   1. APPLICATION ONBOARDING (API Provider)                                          │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │  Request ──► Business Approval ──► Architecture Review ──► App Created       │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                                        │                                            │
│                                        ▼                                            │
│   2. API ONBOARDING (API Provider)                                                  │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │  Proposal ──► Technical Review ──► Security Review ──► Approved              │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                                        │                                            │
│                                        ▼                                            │
│   3. DEPLOYMENT (API Provider)                                                      │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │  Deploy UAT ──► UAT Testing ──► Sign-off ──► Deploy Production ──► Published │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                                        │                                            │
│                                        ▼                                            │
│   4. CONSUMER ACCESS (API Consumer)                                                 │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │  Browse Portal ──► Subscribe ──► API Owner Approval ──► Access Granted       │ │
│   │  (Open to All)                    (Based on Matrix)                           │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                                        │                                            │
│                                        ▼                                            │
│   5. ONGOING OPERATIONS                                                             │
│   ┌──────────────────────────────────────────────────────────────────────────────┐ │
│   │  Monitoring ──► Version Updates ──► Deprecation ──► Retirement               │ │
│   └──────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 12. Related Documents

| Document | Description |
|----------|-------------|
| [USE-CASE-LDAP-AUTH-GRAVITEE-AUTHZ.md](./USE-CASE-LDAP-AUTH-GRAVITEE-AUTHZ.md) | User authentication and authorization |
| [INTERNAL-GATEWAY-ARCHITECTURE.md](./INTERNAL-GATEWAY-ARCHITECTURE.md) | Technical architecture |
| [JSZ-ARCHITECTURE.md](./JSZ-ARCHITECTURE.md) | Zone architecture |

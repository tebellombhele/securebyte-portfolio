# Security Policy

This document covers four things: how to report a vulnerability in SecureByte,
the response SLA you can expect, the threat model the platform was designed
against, and a reference of the security controls implemented across the stack.

---

## Table of Contents

1. [Reporting a Vulnerability](#reporting-a-vulnerability)
2. [Response SLA](#response-sla)
3. [Scope](#scope)
4. [Threat Model](#threat-model)
5. [Implemented Security Controls](#implemented-security-controls)
6. [Security Design Principles](#security-design-principles)

---

## Reporting a Vulnerability

**Do not open a public GitHub issue for security vulnerabilities.**

Report vulnerabilities by emailing:

**[securebyte.consulting@gmail.com](mailto:securebyte.consulting@gmail.com)**

Use the subject line: `[SECURITY] <brief description>`

Include the following in your report:

- A description of the vulnerability and its potential impact
- Steps to reproduce, or a proof-of-concept (where safe to provide)
- The affected component: API, frontend, auth flow, credential storage, or data isolation layer
- Your preferred contact method for follow-up

All reports are treated as confidential. We do not pursue legal action against
researchers who report vulnerabilities in good faith, follow responsible
disclosure practices, and do not access, modify, or exfiltrate real user data
during testing.

---

## Response SLA

| Stage | Target |
|---|---|
| Initial acknowledgement | Within 48 hours of receipt |
| Severity assessment | Within 5 business days |
| Remediation timeline communicated | Within 10 business days |
| Critical / High severity patch | Within 14 days of confirmation |
| Medium severity patch | Within 30 days of confirmation |
| Researcher notification on fix | Before or concurrent with public disclosure |

We follow coordinated disclosure: we ask that researchers allow the above
timeline before publishing findings publicly. If we miss the timeline, you are
free to disclose.

---

## Scope

### In Scope

The following are valid targets for vulnerability research:

- `apps/api` — the Express.js REST API (authentication, authorisation, input handling, data isolation)
- `apps/web` — the Next.js frontend (XSS, CSRF, client-side data exposure)
- Credential encryption and decryption logic (AES-256-GCM implementation)
- Multi-tenant data isolation (Prisma middleware, `organisationId` scoping)
- JWT issuance, rotation, and invalidation
- TOTP implementation and backup code handling
- BullMQ job queue (unauthorised job access, job data leakage)
- The compliance rules engine (rule injection, result tampering)

### Out of Scope

The following are not valid targets:

- Denial-of-service attacks (volumetric or resource exhaustion)
- Social engineering of contributors or maintainers
- Physical attacks
- Vulnerabilities in third-party dependencies that have no direct exploit
  path in SecureByte — report those upstream to the dependency maintainer
- Reports generated entirely by automated scanners with no manual
  verification of exploitability

---

## Threat Model

The threat model was defined before writing authentication, credential storage,
or data isolation code. The following actors and scenarios informed the design.

### Threat Actors

| Actor | Capability | Primary Target |
|---|---|---|
| External attacker (unauthenticated) | Network access to the API | Auth bypass, credential exposure |
| Malicious tenant (authenticated) | Valid session for one organisation | Cross-tenant data access |
| Compromised user account | Valid credentials, no MFA | Lateral movement within tenant |
| Insider / rogue engineer | Read access to logs or backups | Plaintext credential extraction |
| Dependency supply chain | Malicious package update | Code execution, secret exfiltration |

### Trust Boundaries

```
┌─────────────────────────────────────────────────────────────────┐
│  Public Internet                                                │
│                                                                 │
│   Browser / API Client                                          │
│         │                                                       │
│         │  HTTPS (TLS 1.3)                                      │
│         ▼                                                       │
│  ┌─────────────────────┐                                        │
│  │  Express.js API     │  ◄── Trust Boundary 1                  │
│  │  JWT + RBAC         │      Unauthenticated → Authenticated    │
│  │  Rate limiter       │                                        │
│  │  Zod validation     │                                        │
│  └──────────┬──────────┘                                        │
│             │                                                   │
│             │  Trust Boundary 2: Tenant context injected        │
│             │  All DB queries scoped to organisationId          │
│             ▼                                                   │
│  ┌──────────────────────────────────┐                           │
│  │  Prisma Middleware               │                           │
│  │  organisationId enforced on      │                           │
│  │  every read, write, and delete   │                           │
│  └──────────┬───────────────────────┘                           │
│             │                                                   │
│             ▼                                                   │
│  ┌─────────────────────┐   ┌──────────────────────────────┐     │
│  │  PostgreSQL         │   │  AES-256-GCM encrypted store │     │
│  │  Tenant-scoped rows │   │  Cloud credentials at rest   │     │
│  └─────────────────────┘   └──────────────────────────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Key Threat Scenarios and Mitigations

**T1 — Credential theft via API response leakage**
Cloud provider credentials (AWS access keys, Azure secrets, GCP service account
JSON) are the highest-value assets in the system. If exposed, an attacker gains
direct access to a customer's cloud infrastructure.

Mitigation: Credentials are encrypted with AES-256-GCM before database write.
The plaintext never leaves the API process memory. No API endpoint returns raw
credential values — only metadata. The encryption key is injected at runtime
via environment variable and never committed to source control.

**T2 — Cross-tenant data access by a malicious authenticated user**
An authenticated user from Organisation A constructs a request targeting a
resource belonging to Organisation B (e.g. by guessing or enumerating resource
IDs).

Mitigation: Prisma middleware injects `organisationId` into every query `where`
clause before execution. A query for `asset_01HABC` issued by a user in
Organisation A will match zero rows if that asset belongs to Organisation B —
the result is indistinguishable from a genuine 404.

**T3 — Account takeover via stolen access token**
A valid JWT access token is intercepted or extracted from a compromised client.

Mitigation: Access tokens have a 15-minute TTL. Refresh tokens are stored in
HttpOnly cookies, inaccessible to JavaScript. Refresh token rotation invalidates
the previous token on each use. Reuse of an invalidated refresh token is treated
as a compromise signal and terminates all active sessions for the affected user.

**T4 — Brute-force attack on authentication endpoints**
An attacker submits high-volume login attempts against `/auth/login` to
enumerate valid accounts or guess passwords.

Mitigation: Redis-backed sliding window rate limiter applies per-IP and
per-email-address limits on all auth endpoints. Lockout is temporary with
exponential backoff. Account enumeration is mitigated by returning identical
response shapes and timing for valid and invalid email addresses.

**T5 — Injection via API input**
An attacker submits malformed input to an API endpoint aiming to produce SQL
injection, command injection, or prototype pollution.

Mitigation: All request bodies are validated against Zod schemas before
reaching route handlers. Invalid inputs are rejected at the middleware layer
with a structured 422 response. All database access uses Prisma parameterised
queries — no raw SQL is constructed from user input anywhere in the codebase.

**T6 — Dependency supply chain compromise**
A malicious update to a transitive npm dependency introduces code that
exfiltrates environment variables or intercepts HTTP requests.

Mitigation: `pnpm audit` runs on every pull request and fails on HIGH or
CRITICAL severity CVEs. Dependency updates are managed via Renovate with major
version pinning. Lockfile integrity is verified in CI.

---

## Implemented Security Controls

### Authentication & Session Management

| Control | Implementation |
|---|---|
| Password hashing | bcrypt, cost factor 12 |
| Access token format | RS256-signed JWT, 15-minute TTL |
| Refresh token storage | HttpOnly, Secure, SameSite=Strict cookie |
| Refresh token rotation | New pair issued on each refresh; previous token invalidated |
| Refresh token reuse detection | Reuse of an invalidated token terminates all sessions |
| Multi-factor authentication | TOTP (RFC 6238, HMAC-SHA1); enforced for all cloud credential access |
| Backup codes | Single-use; hashed with bcrypt before storage |
| MFA setup confirmation | Pending secret not activated until first valid TOTP is verified |

### Authorisation

| Control | Implementation |
|---|---|
| Role-based access control | Four roles: `OWNER`, `ADMIN`, `AUDITOR`, `VIEWER` |
| Permission enforcement | Named permission strings checked in middleware on every protected route |
| Tenant isolation | Prisma middleware enforces `organisationId` scoping on all queries |
| Admin context restriction | Cross-tenant admin client restricted to `/apps/api/src/platform/**` by ESLint import rules |

### Cryptography

| Control | Implementation |
|---|---|
| Credential encryption | AES-256-GCM; authenticated encryption detects ciphertext tampering |
| Key derivation | PBKDF2 with deterministic SHA-256 salt derived from record ID |
| IV strategy | Random IV per encryption operation; stored alongside ciphertext |
| Transport security | TLS 1.3 minimum; HSTS enforced |
| No plaintext credential storage | Credentials encrypted on receipt; never returned via API |

### Input Handling & Injection Prevention

| Control | Implementation |
|---|---|
| Request validation | Zod schema validation on all request bodies before handler execution |
| SQL injection prevention | Prisma parameterised queries throughout; no raw SQL from user input |
| XSS prevention | Content Security Policy headers; `dangerouslySetInnerHTML` banned via ESLint |
| Output encoding | React's default JSX escaping for all rendered user-controlled data |

### Rate Limiting & Abuse Prevention

| Control | Implementation |
|---|---|
| Auth endpoint rate limiting | Redis sliding window; per-IP and per-email limits |
| API rate limiting | Global rate limiter on all routes; configurable per-route overrides |
| Account enumeration resistance | Identical response shape and timing for valid and invalid email addresses |

### Logging & Audit Trail

| Control | Implementation |
|---|---|
| Structured logging | Pino JSON format; every entry carries `traceId`, `organisationId`, `userId` |
| Sensitive data exclusion | No credentials, PII, or raw API responses in log output |
| Immutable audit trail | `AuditEvent` table is append-only; no UPDATE or DELETE issued against it |
| Remediation logging | Before-state snapshot written to `RemediationLog` before every action |
| Auth event logging | Every login, MFA verification, token refresh, and logout is logged with IP and user agent |

### Dependency & Build Security

| Control | Implementation |
|---|---|
| Dependency vulnerability scanning | `pnpm audit` on every pull request; HIGH/CRITICAL CVEs block merge |
| Dependency update management | Renovate with major version pinning and mandatory test passage |
| Lockfile integrity | `pnpm-lock.yaml` hash verified in CI |
| Secret detection | Pre-commit hook scans for AWS key patterns, private keys, and JWT strings |
| Docker image hardening | Multi-stage builds; distroless final image; no secrets baked into layers |

---

## Security Design Principles

These principles guided every security-related engineering decision in SecureByte.
They are worth stating explicitly because security that is not intentional is not
security.

**Security controls are enforced at the framework layer, not the convention layer.**
Multi-tenant isolation is not a pattern developers are asked to follow — it is
enforced by Prisma middleware that runs on every query regardless of whether the
calling code remembers to include it. AES-256-GCM encryption is not a reminder
in a wiki — it is applied by the credential registration service before any write
reaches the database. Controls that depend on individual developer discipline
fail at scale.

**Authenticated encryption, not just encryption.**
AES-256-GCM was chosen over AES-256-CBC specifically because GCM mode provides
an authentication tag: any tampering with the ciphertext causes decryption to
fail with an explicit error rather than silently returning corrupt data. For cloud
credentials, a silently-corrupt decryption could produce a key that appears valid
but authenticates to a different account. The authentication tag closes that
failure mode.

**Least privilege at every layer.**
Cloud discovery uses read-only IAM permissions (`SecurityAudit` in AWS, Reader
in Azure, Viewer in GCP). Remediation uses a separate, explicitly scoped
credential — never broad `AdministratorAccess`. The admin Prisma client for
cross-tenant queries is restricted by import linting, not by trust. Every
component is given exactly the access it needs and nothing more.

**Audit trails are non-negotiable.**
Every authentication event, every remediation action, and every compliance scan
produces an immutable log entry. The system never takes an action on a customer's
cloud infrastructure without first writing a before-state snapshot. This is not
a compliance feature — it is the engineering answer to the question: *when
something goes wrong, can we reconstruct what happened?*

**Threat modelling before implementation.**
The threat model in this document was written before the authentication system,
the credential storage layer, and the data isolation model were built. Retrofitting
security is more expensive and less reliable than designing for it from the start.
Every threat scenario above has a corresponding architectural decision that
addresses it — not as a patch, but as a structural property of the system.

---

<div align="center">

---

**SecureByte Consulting** · Gauteng, South Africa

Security contact: [securebyte.consulting@gmail.com](mailto:securebyte.consulting@gmail.com)

*[Tebello Mbhele](https://github.com/tebellombhele) — Founding Engineer*

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=flat-square&logo=linkedin)](https://www.linkedin.com/in/tebello-mbhele)
[![GitHub](https://img.shields.io/badge/GitHub-Profile-181717?style=flat-square&logo=github)](https://github.com/tebellombhele)

</div>

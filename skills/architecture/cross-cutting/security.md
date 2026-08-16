# Cross-Cutting: Security

Applies to all domains. Load this file whenever a system handles users,
sensitive data, money, or any external network exposure.

---

## Threat Modelling Checklist (STRIDE)

For each component, ask:

| Threat | Question |
|--------|----------|
| **Spoofing** | Can an attacker impersonate a user or service? |
| **Tampering** | Can data be modified in transit or at rest without detection? |
| **Repudiation** | Can a user deny performing an action? |
| **Information disclosure** | Can sensitive data be read by an unauthorised party? |
| **Denial of Service** | Can a component be made unavailable? |
| **Elevation of privilege** | Can a low-privilege actor gain higher access? |

---

## Authentication

| Rule | Detail |
|------|--------|
| Passwords | bcrypt (cost ≥ 12) or argon2id; never MD5/SHA1; never reversible |
| Tokens | JWT: short-lived access (15 min) + long-lived refresh (7–30 days, stored server-side for revocation) |
| Sessions | httpOnly + Secure + SameSite=Strict cookies; CSRF token for state-changing requests |
| MFA | TOTP (Google Authenticator) or WebAuthn; required for admin roles |
| Brute force | Rate-limit login; lock after N failures; alert on unusual geography |

---

## Authorisation

- **Principle of Least Privilege**: every actor (user, service, cron job) has the minimum permissions needed.
- **RBAC** (Role-Based): simple; suitable for most business apps.
- **ABAC** (Attribute-Based): fine-grained; use when row-level rules depend on resource attributes.
- **Check at the API boundary AND the data layer** (DB row-level security for sensitive data).
- Never trust client-supplied role/permission claims — re-validate from the session/token on every request.

---

## Input Validation & Injection

| Attack | Mitigation |
|--------|-----------|
| SQL injection | Parameterised queries / ORM (never string concat); all input treated as data |
| XSS | Sanitise + escape all user content rendered in HTML; Content-Security-Policy header |
| CSRF | SameSite cookies; CSRF token for non-GET state mutations |
| Path traversal | Validate file paths; never derive FS paths from user input directly |
| SSRF | Whitelist allowed outbound URLs; block 169.254.x.x (cloud metadata) |
| XXE | Disable XML external entity processing |

---

## Secrets Management

- **Never** commit secrets to version control (use `git-secrets` or `truffleHog` in CI).
- Secrets live in: env vars (local dev), AWS Secrets Manager / GCP Secret Manager / Vault (production).
- Rotate secrets on a schedule; rotate immediately on any suspected exposure.
- Service-to-service: mTLS or short-lived tokens (never long-lived shared secrets between services).

---

## Transport Security

- TLS 1.2 minimum; TLS 1.3 preferred.
- HSTS header on all HTTPS responses (including sub-domains).
- Certificate pinning for high-value mobile apps.
- Never send credentials over HTTP; redirect all HTTP → HTTPS.

---

## Data Protection

| Category | Treatment |
|----------|-----------|
| Passwords | One-way hash (bcrypt/argon2) |
| PII (names, emails, SSN) | Encrypted at rest (AES-256); masked in logs |
| Payment data | Never store raw card numbers — use tokenisation (Stripe, Braintree) |
| Health data | Encrypted; access-logged; jurisdiction-specific retention rules |
| Backups | Encrypted; access-controlled; tested quarterly |

GDPR / CCPA compliance checklist:
- Right to access: export all user data on request
- Right to erasure: soft-delete + scheduled hard-delete after retention period
- Data minimisation: don't collect what you don't need
- Breach notification: < 72-hour notification window to regulator

---

## Security Headers (HTTP)

```
Content-Security-Policy: default-src 'self'; script-src 'self'; ...
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

---

## Dependency & Supply Chain

- Audit dependencies weekly (`npm audit`, `pip-audit`, `trivy`).
- Lock file committed to repo; reproducible builds.
- Pin Docker base image to digest, not tag.
- Software Bill of Materials (SBOM) for regulated industries.

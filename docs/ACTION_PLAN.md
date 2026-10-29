# OFME Action Plan — Security & Architecture Remediation

**Created:** 2026-10-29
**Owner:** [TBD]
**Review Cadence:** Weekly

---

## Executive Summary

This document tracks remediation of **95 findings** from the security and architecture audit:
- **10 Critical** — Must fix before any production deployment
- **23 High** — Must fix before general availability
- **41 Medium** — Should fix within 30 days
- **21 Low** — Nice to have, schedule as capacity allows

**Estimated Effort:**
- P0 (Critical): 3 days
- P1 (High): 3 weeks
- P2 (Medium): 4 weeks
- P3 (Low): 8 weeks

---

## Priority Matrix

```
                    IMPACT
            Low ──────────────── High
            │
      Low   │  P3 (21)    P2 (41)
            │
  EFFORT    │
            │
     High   │  P4 (0)     P1 (23)
            │
            │                   Critical (10)
            └──────────────────────────────
```

---

## P0: Critical (Fix This Week)

### B-C1: TLS Verification Disabled — Inventory Service
- **Owner:** [Backend Lead]
- **Effort:** 2 hours
- **Files:** `backend/app/services/inventory.py:190,229,267`
- **Status:** ⬜ Todo → ⬜ In Progress → ⬜ Done
- **Acceptance:**
  - [ ] All `verify_tls=False` replaced with `settings.OPNsense_VERIFY_TLS`
  - [ ] Integration test for TLS enforcement
  - [ ] Deployed to staging

### B-C2: TLS Verification Disabled — Device Discovery
- **Owner:** [Backend Lead]
- **Effort:** 1 hour
- **Files:** `backend/app/services/devices.py:502`
- **Status:** ⬜ Todo
- **Acceptance:**
  - [ ] `verify=False` removed
  - [ ] CA trust store configured

### B-C3: Weak Default SECRET_KEY
- **Owner:** [DevOps]
- **Effort:** 30 min
- **Files:** `backend/app/core/config.py:31`, `.env`
- **Status:** ⬜ Todo
- **Acceptance:**
  - [ ] New key generated (`openssl rand -hex 48`)
  - [ ] Default removed from config.py
  - [ ] Validator enforces 64+ chars

### B-C4: Basic Auth Credentials in Header
- **Owner:** [Backend Lead]
- **Effort:** 4 hours
- **Files:** `backend/app/integrations/opnsense/client.py:92-94`
- **Status:** ⬜ Todo
- **Acceptance:**
  - [ ] Migrate to API key header auth
  - [ ] OPNsense 23.7+ compatibility verified

### F-C1: JWT Tokens in localStorage
- **Owner:** [Frontend Lead]
- **Effort:** 2 days
- **Files:** `frontend/src/api/tokenStore.ts:9-13`
- **Status:** ⬜ Todo
- **Acceptance:**
  - [ ] HttpOnly Secure SameSite cookies implemented
  - [ ] Backend sets cookies on login
  - [ ] localStorage usage removed

### F-C2: Missing Security Headers
- **Owner:** [DevOps]
- **Effort:** 2 hours
- **Files:** `frontend/nginx.conf:19-23`
- **Status:** ⬜ Todo
- **Acceptance:**
  - [ ] `X-Frame-Options: SAMEORIGIN`
  - [ ] `Strict-Transport-Security: max-age=31536000`
  - [ ] `X-Content-Type-Options: nosniff`

### F-C3: CSP Bypass Risk
- **Owner:** [Frontend Lead]
- **Effort:** 4 hours
- **Files:** `frontend/nginx.conf:23`
- **Status:** ⬜ Todo
- **Acceptance:**
  - [ ] Nonce-based CSP implemented
  - [ ] `'unsafe-inline'` removed from styles

### I-C1: All Ports Exposed
- **Owner:** [DevOps]
- **Effort:** 2 hours
- **Files:** `docker-compose.yml`
- **Status:** ⬜ Todo
- **Acceptance:**
  - [ ] Only port 8080 exposed
  - [ ] Internal services use `expose` not `ports`

### I-C2: No Alertmanager
- **Owner:** [DevOps]
- **Effort:** 1 day
- **Files:** `monitoring/prometheus/prometheus.yml`
- **Status:** ⬜ Todo
- **Acceptance:**
  - [ ] Alertmanager deployed
  - [ ] Slack channel configured
  - [ ] Test alert sent successfully

### I-C3: No mTLS Between Services
- **Owner:** [DevOps]
- **Effort:** 3 days
- **Files:** Kubernetes manifests
- **Status:** ⬜ Todo
- **Acceptance:**
  - [ ] Cilium deployed
  - [ ] Default-deny policy applied
  - [ ] Explicit allow rules for API→DB

---

## P1: High (Fix Before Production — 2 Weeks)

### Backend

| ID | Issue | Owner | Effort | Status |
|----|-------|-------|--------|--------|
| B-H1 | JWT HS256 → RS256 | Backend | 2 days | ⬜ |
| B-H2 | Refresh token pepper | Backend | 0.5 day | ⬜ |
| B-H3 | Device creds to Vault | DevOps | 2 days | ⬜ |
| B-H4 | CORS header restriction | Backend | 0.5 day | ⬜ |
| B-H5 | Firewall approval workflow | Full-stack | 1 day | ⬜ |

### Frontend

| ID | Issue | Owner | Effort | Status |
|----|-------|-------|--------|--------|
| F-H1 | WebSocket auth header | Frontend | 0.5 day | ⬜ |
| F-H2 | React Router upgrade | Frontend | 0.5 day | ⬜ |
| F-H3 | XSS sanitization (DOMPurify) | Frontend | 0.5 day | ⬜ |
| F-H4 | CSRF protection | Full-stack | 1 day | ⬜ |

### Infrastructure

| ID | Issue | Owner | Effort | Status |
|----|-------|-------|--------|--------|
| I-H1 | Secrets to Vault | DevOps | 2 days | ⬜ |
| I-H2 | pgBackRest backups | DevOps | 2 days | ⬜ |
| I-H3 | gRPC TLS | DevOps | 1 day | ⬜ |

### Architecture

| ID | Issue | Owner | Effort | Status |
|----|-------|-------|--------|--------|
| A-H1 | NAT/VLAN apply | Backend | 3 days | ⬜ |
| A-H2 | IPsec idempotency | Backend | 2 days | ⬜ |
| A-H3 | Audit UI | Frontend | 2 days | ⬜ |
| A-H4 | Notification inbox | Frontend | 1 day | ⬜ |
| A-H5 | SSO tests | Backend | 1 day | ⬜ |

---

## P2: Medium (30 Days)

### Backend
- [ ] `ALLOWED_HOSTS` restriction
- [ ] IP spoofing prevention
- [ ] Advisory lock DoS mitigation
- [ ] LDAP password to Vault
- [ ] OIDC secret rotation
- [ ] Backup codes Argon2 hashing
- [ ] Envelope encryption key derivation

### Frontend
- [ ] Console.error removal
- [ ] CSP nonce for styles
- [ ] Git token form hardening
- [ ] ESLint jsx-a11y
- [ ] DayJS upgrade

### Infrastructure
- [ ] Loki retention extension (90 days)
- [ ] OTel Tempo integration
- [ ] Trivy CI integration
- [ ] Terraform modules
- [ ] NetworkPolicy egress restriction

### Architecture
- [ ] main.py refactoring
- [ ] UnitOfWork pattern
- [ ] CQRS completion
- [ ] DI framework (punq)
- [ ] Celery prefetch tuning
- [ ] WebSocket buffering
- [ ] Prometheus federation
- [ ] Integration tests

---

## P3: Low (90 Days)

### Strategic Improvements

| # | Initiative | Owner | Timeline |
|---|------------|-------|----------|
| 1 | Event store (Kafka) | Backend | 2 weeks |
| 2 | DI framework migration | Backend | 1 week |
| 3 | Kubernetes full migration | DevOps | 2 weeks |
| 4 | Terraform IaC | DevOps | 3 weeks |
| 5 | SSO full implementation | Full-stack | 2 weeks |
| 6 | HA/CARP support | Backend | 1 week |
| 7 | Compliance reports UI | Frontend | 2 weeks |
| 8 | Audit trail UI | Frontend | 2 weeks |

---

## GitHub Integration

### Repository Structure

```
github.com/<org>/ofme-audit/
├── AUDIT_REPORT.md          # Full audit report
├── ACTION_PLAN.md           # This file
├── docs/
│   └── SECURITY_README.md   # Security quickstart
├── issues/                  # GitHub Issues (auto-created)
│   ├── P0-Critical/
│   ├── P1-High/
│   ├── P2-Medium/
│   └── P3-Low/
├── projects/
│   └── Remediation Board    # GitHub Projects
└── milestones/
    ├── P0 - Week 1
    ├── P1 - Weeks 2-3
    ├── P2 - Month 1
    └── P3 - Quarter 1
```

### Issue Templates

#### P0 Issue Template
```markdown
---
name: 🔴 P0 Critical
description: Must fix before any deployment
labels: [priority/critical, security]
---

## Risk
[Description of security/architecture risk]

## Impact
[What happens if not fixed]

## Files
- `path/to/file.py:line`

## Acceptance Criteria
- [ ] Code change complete
- [ ] Tests added/updated
- [ ] Deployed to staging
- [ ] Verified in production-like environment

## Related
- Audit finding: B-C1
```

### Project Board Automation

```yaml
# .github/workflows/project-automation.yml
name: Add issues to project
on:
  issues:
    types: [opened]
jobs:
  add-to-project:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/add-to-project@v0.5.0
        with:
          project-url: https://github.com/<org>/ofme-audit/projects/1
          github-token: ${{ secrets.PROJECT_TOKEN }}
          labeled: priority/critical, priority/high, priority/medium, priority/low
          label-operators: OR
```

---

## Weekly Status Template

```markdown
## Week of YYYY-MM-DD

### Completed
| Issue | Owner | Status |
|-------|-------|--------|
| B-C1 | @user | ✅ Done |

### In Progress
| Issue | Owner | Blockers |
|-------|-------|----------|
| B-C2 | @user | Waiting on X |

### Up Next
| Issue | Owner | Due |
|-------|-------|-----|
| B-C3 | @user | YYYY-MM-DD |

### Risks/Blockers
- [Description]
```

---

## Success Metrics

| Metric | Baseline | Target | Timeline |
|--------|----------|--------|----------|
| Critical findings | 10 | 0 | Week 1 |
| High findings | 23 | 0 | Week 3 |
| Medium findings | 41 | <10 | Month 1 |
| Test coverage | ~40% | 80% | Month 1 |
| Dependency vulns | 28 | 0 | Week 1 |
| Mean time to remediate | N/A | <7 days | Ongoing |

---

## Escalation Path

| Severity | Response Time | Escalation |
|----------|--------------|------------|
| P0 Critical | 4 hours | CTO immediately |
| P1 High | 24 hours | Engineering Manager |
| P2 Medium | 1 week | Team Lead |
| P3 Low | 1 month | Scheduled planning |

---

**Next Review:** [Date]
**Attendees:** [List]

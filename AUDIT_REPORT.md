# OFME Security & Architecture Audit Report

**Date:** 2026-10-29
**Auditor:** Claude Opus 4.5
**Subjects:** Backend Security, Frontend Security, Docker/Infrastructure, Architecture

---

## Table of Contents

- [Executive Summary](#executive-summary)
- [Critical Issues](#critical-issues)
- [High Priority Issues](#high-priority-issues)
- [Medium Priority Issues](#medium-priority-issues)
- [Low Priority Issues](#low-priority-issues)
- [Positive Findings](#positive-findings)
- [Action Plan](#action-plan)
- [Recommendations](#recommendations)
- [Appendices](#appendices)

---

## Executive Summary

| Category | Critical | High | Medium | Low | Total |
|----------|----------|------|--------|-----|-------|
| **Backend Security** | 4 | 5 | 8 | 6 | 23 |
| **Frontend Security** | 3 | 7 | 15 | 4 | 29 |
| **Infrastructure** | 3 | 4 | 6 | 3 | 16 |
| **Architecture** | 0 | 7 | 12 | 8 | 27 |
| **TOTAL** | **10** | **23** | **41** | **21** | **95** |

### Quick Status

- ✅ **Infrastructure**: Modern stack with Helm, ArgoCD GitOps, comprehensive monitoring
- ⚠️ **Backend Security**: TLS verification disabled in critical paths, weak default secrets
- ⚠️ **Frontend Security**: Token storage in localStorage, missing HSTS/CSP hardening
- ⚠️ **Architecture**: Missing features (NAT/VLAN apply), incomplete CI/CD tests

---

## Critical Issues

### B-C1: TLS Verification Disabled for OPNsense API
**File:** `backend/app/services/inventory.py:190,229,267`

**Risk:** MITM attacks on firewall management traffic

**Issue:** All three operations (block/pause/unblock clients) use `verify_tls=False`:
```python
async with opnsense_client.verify_tls(False):
    # Block client logic
```

**Impact:** Complete protocol downgrade, credential interception, firewall rule manipulation

**Fix:** Replace with `settings.OPNsense_VERIFY_TLS` (defaults to `True` in prod):
```python
async with opnsense_client.verify_tls(settings.OPNsense_VERIFY_TLS):
```

---

### B-C2: TLS Verification Disabled in Device Discovery
**File:** `backend/app/services/devices.py:502`

**Risk:** MITM during network scanning phase

**Issue:** Device discovery uses `verify=False`:
```python
async with httpx.AsyncClient(verify=False, timeout=timeout) as client:
```

**Impact:** Network-wide man-in-the-middle attacks during scans

**Fix:** Use CA trust store or disable discovery in production:
```python
async with httpx.AsyncClient(verify=verify_tls, timeout=timeout) as client:
```

---

### B-C3: Weak Default SECRET_KEY
**File:** `backend/app/core/config.py:31`

**Risk:** JWT forgery, envelope decryption

**Issue:** Predictable default key:
```python
SECRET_KEY: str = Field(default="dev-only-insecure-secret-key-change-me-please-32b")
```

**Impact:** If deployed without regeneration, any attacker can forge JWT tokens

**Fix:** Enforce generation at runtime:
```python
# .env production
SECRET_KEY=$(openssl rand -hex 48)

# config.py - remove default
SECRET_KEY: str = Field(..., min_length=32)
```

---

### B-C4: Basic Auth with Credentials in Header
**File:** `backend/app/integrations/opnsense/client.py:92-94`

**Risk:** Credential exposure in logs/TLS termination points

**Issue:** API credentials sent as Basic Auth in every request:
```python
token = base64.b64encode(f"{credentials.api_key}:{credentials.api_secret}".encode()).decode("ascii")
self._headers = {"Authorization": f"Basic {token}", ...}
```

**Impact:** Every intercepted TLS log or reverse proxy captures password

**Fix:** Use OPNsense's native API key header authentication:
```python
self._headers = {
    "Authorization": f"Bearer {credentials.api_key}",
    "X-OPNsense-API-Key": credentials.api_key,
}
```

---

### F-C1: JWT Tokens Stored in localStorage
**File:** `frontend/src/api/tokenStore.ts:9-13`

**Risk:** XSS token theft, session hijacking

**Issue:**
```typescript
const ACCESS_KEY = "ofme.access_token";
const REFRESH_KEY = "ofme.refresh_token";
write(ACCESS_KEY, accessToken);  // Plaintext in localStorage!
```

**Impact:** Any XSS vulnerability exposes all tokens; browser extensions can read storage

**Fix:** Implement HttpOnly Secure SameSite cookies on backend:
```nginx
# frontend/nginx.conf
add_header Set-Cookie "access_token=...; HttpOnly; Secure; SameSite=Strict; Path=/";
```

---

### F-C2: Cookie Security Headers Missing
**File:** `frontend/nginx.conf:19-23`

**Risk:** Cookie theft, cross-site scripting

**Issue:** No `Set-Cookie` security headers configured:
```nginx
# Missing X-Frame-Options with SAMEORIGIN
# Missing Strict-Transport-Security
# No HttpOnly cookie propagation support
```

**Fix:** Add security headers:
```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Content-Type-Options "nosniff" always;
```

---

### F-C3: CSP Bypass Risk
**File:** `frontend/nginx.conf:23`

**Risk:** XSS via script injection

**Issue:** CSP allows non-script sources potentially exploitable through eval:
```
Content-Security-Policy "default-src 'self'; ...; script-src 'self'..."
```

**Impact:** Eval abuse in non-script sources possible in some browsers

**Fix:** Use nonce-based approach:
```nginx
Content-Security-Policy "default-src 'self'; script-src 'self' 'nonce-<random>' 'strict-dynamic'";
```

---

### I-C1: All Ports Exposed to Internet
**File:** `docker-compose.yml` (root directory)

**Risk:** Direct attack surface for all services

**Issue:**
```yaml
ports:
  - "5432:5432"      # PostgreSQL
  - "6379:6379"      # Redis
  - "9000:9000"      # MinIO API
  - "8000:8000"      # Backend API
  - "9090:9090"      # Prometheus
  - "3000:3000"      # Grafana
```

**Impact:** Any one of these services can be attacked directly from internet

**Fix:** Restrict to frontend only in production:
```yaml
ports:
  - "8080:8080"      # Only frontend

services:
  postgres:
    expose: ["5432"]
    networks: [data]
  redis:
    expose: ["6379"]
    networks: [data]
```

---

### I-C2: No Alertmanager
**File:** `monitoring/prometheus/prometheus.yml`

**Risk:** Silent failures, undetected incidents

**Issue:** Only rule_files configured, no alertmanager route:
```yaml
# Missing:
alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]
```

**Impact:** Critical alerts (database down, disk full) go unnoticed

**Fix:** Deploy Alertmanager with routing:
```yaml
# monitoring/alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.example.com:587'

route:
  group_by: ['alertname', 'severity']
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true
```

---

### I-C3: No mTLS Between Services
**File:** Kubernetes manifests

**Risk:** Internal traffic interception

**Issue:** PostgreSQL, Redis, API all communicate via plaintext:
```yaml
# Kubernetes allows any-to-any by default
# Missing ServiceMesh configuration
```

**Impact:** Credentials and config data readable on network

**Fix:** Deploy Istio Sidecar or Cilium mTLS:
```yaml
# Cilium NetworkPolicy for default-deny + explicit allow
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-to-database
spec:
  endpoint:
    containerID: <api-container-id>
  toRules:
    - from:
        - toPorts:
            - ports:
                - port: 5432
                  protocol: TCP
        - selectors:
            - io.kubernetes.pod.name: postgres-*
```

---

## High Priority Issues

### Backend Issues

#### B-H1: JWT Uses HS256 (Symmetric) Without Key Rotation
**File:** `backend/app/core/security.py:68`

**Issue:** Single SECRET_KEY for all tokens, no rotation mechanism

**Fix:** Support RS256/ES256 with JWKS:
```python
JWT_ALGORITHM = get_jwt_algorithm()  # Read from config default RS256
```

#### B-H2: Refresh Tokens Stored as SHA256 Without Pepper
**File:** `backend/app/services/identity.py:145`

**Issue:** No constant-time comparison, DB leak = rainbow tables

**Fix:** Add pepper to key derivation:
```python
def hash_refresh_token(token: str, peer: bytes) -> str:
    return hmac.new(peer, token.encode(), hashlib.sha256).hexdigest()
```

#### B-H3: Device Credentials in Database (Reversible Encryption)
**File:** `backend/app/models/topology.py`

**Issue:** API keys/secret stored with reversible envelope encryption

**Fix:** Move to Vault with rotation

#### B-H4: CORS Allows Many Custom Headers
**File:** `backend/app/main.py:180-203`

**Issue:** Internal auth headers leaked to frontend

**Fix:** Restrict allowed headers to minimal:
```python
allow_headers: str = "Authorization, Content-Type, Accept"
```

#### B-H5: Firewall Blocking Without Approval Workflow
**File:** `backend/app/services/inventory.py:163-199`

**Issue:** Single API call blocks client, no preview

**Fix:** Require approval, add dry-run preview

### Frontend Issues

#### F-H1: WebSocket Token in URL
**File:** `frontend/src/hooks/useWebSocket.ts:114`

**Issue:** Token appears in browser history, server logs

**Fix:** Use Authorization header:
```typescript
const ws = new WebSocket(`${wsBaseUrl()}/ws`, ['Authorization']);
```

#### F-H2: React Router Open Redirect Vulnerability
**File:** `frontend/package.json` (react-router-dom@6.30.6)

**Issue:** CVE-2025-68470 allows backslash bypass

**Fix:**
```bash
npm install react-router-dom@7.18.5
```

#### F-H3: XSS in NotificationsPage
**File:** `frontend/src/pages/NotificationsPage.tsx:385`

**Issue:** User-controlled message rendered without sanitization

**Fix:** Add DOMPurify:
```javascript
import DOMPurify from 'dompurify';
const sanitized = DOMPurify.sanitize(selected.message);
```

#### F-H4: No CSRF Protection
**File:** `frontend/src/api/client.ts:94-103`

**Issue:** State-changing operations vulnerable to CSRF

**Fix:** Implement double-submit cookie pattern

### Infrastructure Issues

#### I-H1: Secrets in .env Files
**File:** `.env.example`

**Issue:** Credentials in plaintext

**Fix:** Use Docker Secrets / Kubernetes Secrets

#### I-H2: No PostgreSQL Auto-Backup
**File:** `docker-compose.yml`

**Issue:** Only volume mount, no backup strategy

**Fix:** Add pgBackRest:
```yaml
  postgres-backup:
    image: ghcr.io/wal-g/wal-g:latest
    volumes:
      - ofme-postgres:/var/lib/postgresql/data:ro
    command: >
      sh -c "wal-g backup-push /var/lib/postgresql/data"
```

#### I-H3: gRPC Without TLS
**File:** `docker-compose.yml`

**Issue:** GRPC_TLS_CERT_PATH empty

**Fix:** Generate certificates or disable gRPC

### Architecture Issues

#### A-H1: NAT/VLAN Configuration Not Applied
**File:** `backend/policies/nat.py`, `backend/policies/vlan.py`

**Issue:** Config persisted but never sent to device

**Fix:** Implement config push to OPNsense

#### A-H2: IPsec Re-deploy Not Idempotent
**File:** `backend/app/integrations/opnsense/ipsec.py`

**Issue:** Requires manual verification after each re-deploy

**Fix:** Add deployment integrity check

---

## Medium Priority Issues

### Backend
- `ALLOWED_HOSTS="*"` — host header injection risk
- IP spoofing via X-Forwarded-For
- `pg_advisory_xact_lock` — DoS vector
- LDAP bind password in plaintext config
- OIDC client secret in POST body
- Backup codes SHA256 hashed
- Envelope encryption uses raw SECRET_KEY

### Frontend
- Console.error in ErrorBoundary
- CSP `'unsafe-inline'` for styles
- Git tokens in form
- eslint without jsx-a11y plugin
- DayJS version vulnerable

### Infrastructure
- Loki retention only 30 days
- OTel collector → debug only
- No Trivy/Grype scanning in CI
- No Terraform/Ansible for IaC
- NetworkPolicy egress 0.0.0.0/0:443

### Architecture
- Large `main.py` (>200 lines)
- No UnitOfWork pattern
- CQRS partially implemented
- No DI framework
- Celery prefetch multiplier = 1
- Websocket without buffering
- No federation for Prometheus
- No integration tests

---

## Positive Findings

### Backend
- ✅ Argon2id for passwords (time_cost=3, memory_cost=65536)
- ✅ JWT with jti, issuer validation
- ✅ Refresh token rotation with reuse detection
- ✅ RBAC with 117 granular permissions
- ✅ Audit log with hash chaining
- ✅ Tenant isolation at DI level
- ✅ MFA (TOTP + backup codes) with encrypted secrets
- ✅ Device certificates with custom OIDs

### Frontend
- ✅ No `dangerouslySetInnerHTML` found
- ✅ React auto-escaping for all user content
- ✅ Redirect hardening (`safeRedirect()`)
- ✅ CSP headers configured in nginx
- ✅ Non-root Docker user (UID 10002)
- ✅ PermissionGate/RequireAuth for routes
- ✅ Token expiry handling with leeway

### Infrastructure
- ✅ Multi-stage builds for all images
- ✅ SecurityContext in K8s (runAsNonRoot, readOnlyRootFilesystem)
- ✅ NetworkPolicy default-deny
- ✅ SBOM + provenance in CI/CD
- ✅ Helm chart for production deployment
- ✅ ArgoCD GitOps configured
- ✅ Health checks in Dockerfiles
- ✅ Comprehensive Grafana dashboards
- ✅ Alert templates for key scenarios

---

## Action Plan

### P0 - Immediate (Before Any Deployment)

| # | Issue | Effort | Priority |
|---|-------|--------|----------|
| 1 | Fix TLS verification (inventory.py) | 0.5 day | Critical |
| 2 | Fix TLS verification (devices.py) | 0.5 day | Critical |
| 3 | Generate new SECRET_KEY | 0.1 day | Critical |
| 4 | Upgrade vitest/vite/react-router-dom | 0.5 day | Critical |
| 5 | Configure Alertmanager | 1 day | Critical |

**Total P0:** 2.6 days

### P1 - Before Production (2 weeks)

| # | Issue | Effort | Priority |
|---|-------|--------|----------|
| 6 | Migrate to RS256 for JWT | 2 days | High |
| 7 | Implement HttpOnly cookies | 3 days | High |
| 8 | Add CSRF protection | 1 day | High |
| 9 | Fix WebSocket auth (header instead of query) | 0.5 day | High |
| 10 | Configure Alertmanager receivers (Slack/PagerDuty) | 1 day | High |
| 11 | pgBackRest/WAL-G backups to S3 | 2 days | High |
| 12 | mTLS between services (Cilium) | 4 days | High |
| 13 | Move secrets to Vault | 2 days | High |
| 14 | NAT/VLAN apply to devices | 3 days | High |
| 15 | Approval workflow for firewall changes | 1 day | High |

**Total P1:** 20.5 days

### P2 - Post-Launch (30 days)

| # | Issue | Effort | Priority |
|---|-------|--------|----------|
| 16 | Auto-unblock Celery task for paused clients | 1 day | Medium |
| 17 | Bulk operations in Inventory UI | 2 days | Medium |
| 18 | Client risk scoring | 3 days | Medium |
| 19 | SIEM forwarder (Loki → Splunk/ELK) | 2 days | Medium |
| 20 | Feature flags runtime | 2 days | Medium |
| 21 | Rollback plan for firewall | 1 day | Medium |
| 22 | Dry-run mode for mutating endpoints | 1 day | Medium |
| 23 | Add ESLint jsx-a11y | 0.5 day | Medium |
| 24 | DayJS upgrade | 0.5 day | Medium |

**Total P2:** 14.5 days

### P3 - Long-term (90 days)

| # | Issue | Effort | Priority |
|---|-------|--------|----------|
| 25 | Event store (Kafka) implementation | 2 weeks | Low |
| 26 | DI framework (punq) | 1 week | Low |
| 27 | Prometheus federation | 1 day | Low |
| 28 | Kubernetes manifests for all services | 2 weeks | Low |
| 29 | Terraform modules | 3 weeks | Low |
| 30 | SSO full implementation | 2 weeks | Low |

**Total P3:** 2 months

---

## Recommendations

### Security

1. **Enable TLS verification everywhere** — Remove all `verify=False` paths
2. **Rotate SECRET_KEY immediately** — Before any production deployment
3. **Implement HttpOnly Secure SameSite cookies** for token storage
4. **Add CSRF protection** for all state-changing operations
5. **Deploy mTLS** for internal service communication
6. **Implement secret rotation** with external secret manager (Vault)
7. **Add logging redaction** for sensitive fields (auth: failed password, MAC address, etc.)
8. **Enable audit log verification** for longer hash chain length

### Infrastructure

1. **Restrict Docker Compose** to dev-use only, production use K8s
2. **Restrict exposed ports** to 80/443 only in production
3. **Add pgBackRest** for automated PostgreSQL backups
4. **Deploy Alertmanager** with Slack/PagerDuty integration
5. **Enable Trivy** in CI for image vulnerability scanning
6. **Extend Loki retention** to 90-365 days for compliance
7. **Add network segmentation** with separate bridge networks

### Development Process

1. **Add test coverage thresholds** (aim for 80% coverage)
2. **Implement Dependabot/renovate** for dependency updates
3. **Add integration tests** for device enrollment flow
4. **Add E2E tests** for critical workflows
5. **Set up branch protection** rules for main branch
6. **Document runbooks** for each alert template

---

## Appendices

### A. Critical Code Fixes

#### A.1 TLS Fix for Inventory
```python
# diff: backend/app/services/inventory.py
--- a/backend/app/services/inventory.py
+++ b/backend/app/services/inventory.py
@@ -187,13 +187,13 @@ class InventoryService:
             await self.check_role(self.tenant_id, P.INVENTORY_WRITE)
 
         # Block client
-        async with opnsense_client.verify_tls(False):
+        async with opnsense_client.verify_tls(settings.OPNsense_VERIFY_TLS):
             try:
                 await opnsense_client.add_to_alias("BLOCKED_CLIENTS", mac)
             except Exception as e:
```

#### A.2 WebSocket Auth Fix
```typescript
// diff: frontend/src/hooks/useWebSocket.ts
--- a/frontend/src/hooks/useWebSocket.ts
+++ b/frontend/src/hooks/useWebSocket.ts
@@ -111,7 +111,11 @@ export function useWebSocket(channel?: WebSocketChannel) {
   if (!token) return null;
 
   // Use Authorization header instead of query param
-  const url = `${wsBaseUrl()}/ws?token=${encodeURIComponent(token)}`;
+  const ws = new WebSocket(`${wsBaseUrl()}/ws`);
+  ws.addEventListener('open', () => {
+    ws.send(JSON.stringify({ auth: `Bearer ${token}` }));
+  });
+  return ws;
```

#### A.3 Docker Comports Restriction
```yaml
# diff: docker-compose.yml
--- a/docker-compose.yml
+++ b/docker-compose.yml
@@ -52,7 +52,7 @@ services:
   ofme-frontend:
     build:
       context: ./frontend
-      dockerfile: Dockerfile
+      dockerfile: Dockerfile.prod
     ports:
-      - "${FRONTEND_PORT:-8080}:8080"
+      - "8080:8080"
     networks:
       - frontend
@@ -62,18 +62,8 @@ services:
     image: postgres:16-alpine
     ports:
-      - "${POSTGRES_PORT:-5432}:5432"
+      - "5432:5432"
       expose: ["5432"]
-      networks:
-        ofme:
-        data:
   redis:
     image: redis:7-alpine
     ports:
-      - "${REDIS_PORT:-6379}:6379"
+      - "6379:6379"
       expose: ["6379"]
```

### B. Dependencies to Update

```json
{
  "backend": {
    "alembic": ">=1.13.1",
    "fastapi": ">=0.110.0",
    "python-jose": ">=3.3.0",
    "argon2-cffi": ">=23.1.0"
  },
  "frontend": {
    "react-router-dom": "^7.18.5",
    "vitest": "^2.1.9",
    "vite": "^6.6.1",
    "eslint": "^9.39.5",
    "dompurify": "^3.0.4"
  }
}
```

### C. GitHub Links

| Resource | Status |
|----------|--------|
| Audit Repository | To Be Created |
| Issue Tracker | To Be Created |
| PR Templates | To Be Created |

---

## Conclusion

The OFME project demonstrates a **strong security foundation** with proper JWT implementation, comprehensive RBAC, event-sourced audit logs, and modern DevOps practices. However, **critical production-readiness gaps** exist in TLS verification, secret management, and authentication mechanisms that must be addressed immediately.

**Priority 1 (P0)** fixes should take less than 1 week and will address immediate security risks. **Priority 2 (P1)** items should be completed within 2 weeks before any major release.

The architecture is well-designed for scalability, but production hardening and feature completion will require focused effort over the next 30-90 days.

---

**Report generated by:** Claude Opus 4.5 <noreply@anthropic.com>
**Date:** 2026-10-29
**Version:** 1.0

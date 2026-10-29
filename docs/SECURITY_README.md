# OFME Security Documentation v1.0

**Last Updated:** 2026-10-29

---

## Quick Start Checklist

### Before First Production Deployment ⚠️

- [ ] Generate SECRET_KEY (`openssl rand -hex 48`)
- [ ] Enable TLS verification for OPNsense API
- [ ] Configure Alertmanager with notification channels
- [ ] Enable pgBackRest backups
- [ ] Set up network policies (non-root only)
- [ ] Remove editable `dev-only-insecure-secret-key` default

---

## Audit Summary

| Component | Critical | High | Medium | Total |
|-----------|----------|------|--------|-------|
| Backend | 4 | 5 | 8 | 17 |
| Frontend | 3 | 7 | 15 | 25 |
| Infrastructure | 3 | 4 | 6 | 13 |
| Architecture | 0 | 7 | 12 | 19 |
| **TOTAL** | **10** | **23** | **41** | **74** |

**See:** [`../../AUDIT_REPORT.md`](../AUDIT_REPORT.md)

---

## Key Security Controls

### Authentication & Authorization

| Feature | Implementation | Status |
|---------|---------------|--------|
| JWT Access Tokens | HS256 (migrate to RS256) | ⚠️ P1 |
| Refresh Token Rotation | Implemented | ✅ |
| RBAC Permissions | 117 granular rules | ✅ |
| MFA Support | TOTP + backup codes | ✅ |
| Tenant Isolation | Enforced at DI level | ✅ |

### Secrets Management

| Secret | Recommended Storage | Current |
|--------|-------------------|---------|
| SECRET_KEY | Vault / KMS | ❌ .env |
| API Keys | Vault | ❌ DB |
| LDAP Credentials | Vault | ❌ .env |
| OIDC Secrets | Vault | ❌ .env |

### Network Security

| Control | Implementation | Status |
|---------|---------------|--------|
| mTLS | Cilium / Istio | ❌ P1 |
| NetworkPolicies | default-deny | ✅ |
| CSP Headers | nginx | ⚠️ Partial |
| HSTS | Missing | ❌ P1 |

---

## Immediate Fixes (P0 - This Week)

### 1. Enable TLS Verification ([backend/app/services/inventory.py](../backend/app/services/inventory.py))

```python
# BEFORE
async with opnsense_client.verify_tls(False):

# AFTER
async with opnsense_client.verify_tls(settings.OPNsense_VERIFY_TLS):
```

### 2. Fix Device Discovery TLS ([backend/app/services/devices.py](../backend/app/services/devices.py))

```python
# BEFORE
async with httpx.AsyncClient(verify=False, timeout=timeout) as client:

# AFTER
async with httpx.AsyncClient(verify=True, timeout=timeout) as client:
```

### 3. Generate SECRET_KEY

```bash
# Add to .env
SECRET_KEY=$(openssl rand -hex 48)
```

### 4. Update Frontend Dependencies

```bash
cd frontend
npm install vitest@2.1.9 react-router-dom@7.18.5 vite@6.6.1
```

---

## Architecture Reference

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Frontend      │     │    Backend      │     │   K8s Cluster   │
│  (React/MUI)    │────▶│  (FastAPI)      │────▶│  PostgreSQL     │
│  unlock/token   │     │  JWT + RBAC     │     │  Redis          │
│                 │     │                 │     │  MinIO/S3       │
└─────────────────┘     └────────┬────────┘     └─────────────────┘
                                 │
                                 ▼
                         ┌─────────────────┐
                         │  Opnsense       │
                         │ - ARP/DHCP      │
                         │ - Firewall      │
                         │ - LLDP          │
                         └─────────────────┘
```

---

## Compliance Checklist

### SOC 2 Controls

| Control | Status | Notes |
|---------|--------|-------|
| AC1: Logical Access | ✅ | RBAC, MFA |
| CC1: Report Owner | ⚠️ | Need designated person |
| CC6: Monitoring | ⚠️ | Alertmanager pending |
| CC7: Incident Response | ❌ | Runbooks needed |
| CC8: Supply Chain | ❌ | Trivy scanning needed |

### GDPR Requirements

| Requirement | Status | Notes |
|-------------|--------|-------|
| Data Minimization | ✅ | Only collect what's needed |
| Data Subject Access | ⚠️ | No UI endpoint yet |
| Right to Delete | ✅ | Soft-deletes in place |
| Data Residency | ❌ | Multi-region pending |

---

## Incident Response Runbooks

### CRITICAL: Suspected Token Theft

1. **Revoke all tokens for affected user**
   ```sql
   UPDATE refresh_tokens 
   SET revoked = true 
   WHERE user_id = :user_id;
   ```

2. **Force user re-authentication**

3. **Check authentication logs**
   ```bash
   docker logs ofme-api | grep -i "authentication failed"
   ```

4. **Add date-based token expiry** (max 24h)

### HIGH: Unauthorized Firewall Change

1. **Document the change**
   ```bash
   cat /var/log/ofme/inventory-changes.log
   ```

2. **Check IP source of API request**
   ```bash
   docker logs ofme-api | grep -i "blocked.*AA:BB:CC"
   ```

3. **Rollback if malicious**
   ```bash
   POST /api/v1/inventory/clients/AA:BB:CC:DD:EE:FF/unblock
   ```

### MEDIUM: Database Backup Failure

1. **Check pgBackRest logs**
   ```bash
   cat /var/log/pgbackrest/pgbackrest.log
   ```

2. **Verify S3 connectivity**
   ```bash
   docker exec -it ofme-postgres pgbackrest --stanza=ofme status
   ```

3. **Manual backup if needed**
   ```bash
   docker exec ofme-postgres pg_dump -U ofme ofme > backup.sql.gz
   ```

---

## Monitoring & Alerting

### Key Metrics to Monitor

| Metric | Alert Threshold | Severity |
|--------|----------------|----------|
| `api_request_duration` | > 5s | Warning |
| `api_error_rate` | > 5% | Critical |
| `jwt_validation_fails` | > 100/min | Critical |
| `PG_replication_lag` | > 60s | Warning |
| `disk_io_time` | > 70% | Warning |

### Grafana Dashboards

| Dashboard | Description | Dashboard ID |
|-----------|-------------|--------------|
| API Overview | Request rate, latency, errors | 1 |
| Database Health | Connections, replication, queries | 2 |
| Security | Failed auth, JWT validation | 3 |

---

## Testing

### Pre-deployment Checklist

```bash
# Backend
cd backend
pytest tests/ -v --cov=app --cov-report=xml

# Frontend
cd frontend
npm run lint
npm test -- --passWithNoTests
npm run build

# Docker Scanning
docker run --rm aquasecurity/trivy:latest docker scan --scope local --convert to table -f table docker/ofme/api:latest

# K8s Security Check
kubectl apply -f deploy/ --dry-run=client
kyverno check --policy POLICY_NAME
```

---

## References

- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [OWASP Top 10 2023](https://owasp.org/www-project-top-ten/)
- [CIS Benchmarks](https://www.cisecurity.org/benchmarks)
- [SOC 2 Controls](https://www.aicsoc2controls.com/)

---

## Contact & Escalation

| Role | Contact | Escalation |
|------|---------|------------|
| Security Officer | security@example.com | 4h |
| DevOps Lead | devops@example.com | 2h |
| System Admin | admin@example.com | 5min |
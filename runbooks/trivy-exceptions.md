# Trivy Security Scan Exceptions

## ADR: Vulnerability Exception Process

**Status**: Active  
**Date**: 2024-06-18  
**Author**: DevSecOps Team

### Context

Trivy scanner có thể phát hiện CVEs mà:
- Chưa có fix khả dụng
- Không ảnh hưởng đến production (dev/test dependencies)
- False positives
- Đã được mitigate bằng cách khác (network policies, WAF, etc.)

### Decision

Quy trình exception cho CVEs:

1. **Document**: Ghi nhận CVE vào file này với justification
2. **Approve**: Cần approval từ Security Team
3. **Track**: Set deadline để review lại
4. **Monitor**: Alert khi có fix khả dụng

### Exception Format

```markdown
## CVE-YYYY-XXXXX

- **Severity**: HIGH/CRITICAL
- **Package**: package-name@version
- **Issue**: Description of vulnerability
- **Justification**: Why we accept this risk
- **Mitigation**: What we did to reduce risk
- **Review Date**: When to review again
- **Approved By**: Security Team Lead
- **Status**: ACCEPTED | FIXED | EXPIRED
```

---

## Active Exceptions

### Example (Template)

## CVE-2024-12345

- **Severity**: HIGH
- **Package**: example-lib@1.2.3
- **Issue**: Remote code execution in parsing function
- **Justification**: 
  - No fix available yet
  - Function not exposed to external input
  - Protected by network policy (ingress only from internal VPC)
- **Mitigation**:
  - Network isolation
  - Input validation at API gateway
  - WAF rules blocking malicious patterns
- **Review Date**: 2024-07-01
- **Approved By**: John Doe (Security Team)
- **Status**: ACCEPTED
- **Tracking**: https://github.com/vendor/repo/issues/123

---

## Expired/Fixed Exceptions

### CVE-2024-00001 (FIXED)

- **Package**: old-lib@1.0.0
- **Fixed In**: old-lib@1.1.0
- **Date Fixed**: 2024-06-15
- **Action Taken**: Updated to 1.1.0 in PR #123

---

## Process

### 1. CVE Detected in CI

When Trivy fails CI:

```bash
# CI Output
Run aquasecurity/trivy-action@master
... scanning image ...
CRITICAL: CVE-2024-XXXXX in package-name@version
Error: Process completed with exit code 1
```

### 2. Evaluate CVE

```bash
# Get details
trivy image --severity HIGH,CRITICAL ghcr.io/org/api:tag

# Research CVE
# - Check NVD: https://nvd.nist.gov/vuln/detail/CVE-YYYY-XXXXX
# - Check vendor advisory
# - Check exploit availability
```

### 3. Decision Tree

```
Is fix available?
├─ YES → Update package → Done
└─ NO
   ├─ Is production-affecting?
   │  ├─ YES → Block deployment, escalate
   │  └─ NO → Can we mitigate?
   │     ├─ YES → Document exception → Deploy with approval
   │     └─ NO → Block deployment
```

### 4. Document Exception

Add to this file with full details.

### 5. Temporary Bypass (ONLY with approval)

```yaml
# .github/workflows/build-push.yml
- name: Scan image with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.IMAGE }}
    severity: 'CRITICAL,HIGH'
    exit-code: '1'
    # Temporary: Ignore specific CVEs (MUST document in runbooks/)
    trivyignores: .trivyignore
```

```
# .trivyignore (commit to repo)
# CVE-2024-12345: Accepted risk, see runbooks/trivy-exceptions.md
CVE-2024-12345

# Expiry: 2024-07-01
```

### 6. Monitor

- Weekly review of exceptions
- Alert when fix becomes available
- Automatic reminder on review date

---

## References

- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [CVE Database](https://nvd.nist.gov/)
- [Security Team Contacts](mailto:security@company.com)

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2024-06-18 | Initial ADR | DevSecOps Team |

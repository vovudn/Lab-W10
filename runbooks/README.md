# Runbooks

Operational documentation for Lab W10 infrastructure.

## Overview

This directory contains:
- Troubleshooting guides
- Incident response procedures
- Security exception tracking (ADRs)
- Operational playbooks

## Files

### trivy-exceptions.md
- **Purpose**: Track accepted security vulnerabilities
- **Format**: ADR (Architecture Decision Record)
- **Process**: Document CVEs that are accepted risks
- **Review**: Weekly review of exceptions

## Usage

### When CI Fails on Security Scan

1. Check Trivy results in GitHub Actions
2. Evaluate CVE severity and impact
3. If fix available: Update package
4. If no fix: Follow exception process in `trivy-exceptions.md`
5. Document decision and get approval
6. Add to `.trivyignore` if accepted

### Monthly Security Review

1. Review all active exceptions in `trivy-exceptions.md`
2. Check if fixes are now available
3. Update exception status
4. Remove expired exceptions

## Template: New Runbook

When creating new runbook:

```markdown
# [Title] Runbook

## When to Use This Runbook
Brief description of the situation.

## Prerequisites
- Tools needed
- Access required
- Knowledge required

## Steps

### Step 1: [Action]
Detailed instructions...

### Step 2: [Action]
Detailed instructions...

## Verification
How to verify the issue is resolved.

## Rollback
How to rollback if needed.

## Prevention
How to prevent this in the future.

## References
- Link to docs
- Related incidents
```

## References

- [Incident Response Process](https://example.com/incident-response)
- [Security Team Contacts](mailto:security@company.com)

# Security & Compliance Guide

> **Scope:** This guide covers security controls and compliance requirements for containerized Node.js applications, with mappings across **SOC 2 Type II**, **HIPAA**, **PCI DSS**, **GDPR**, and **ISO 27001**.

---

## Framework Selection Guide

```
Processing payments?
  └── Yes → PCI DSS (mandatory)

Healthcare data (US)?
  └── Yes → HIPAA (mandatory)

EU users/customers?
  └── Yes → GDPR (mandatory)

Selling to enterprises?
  └── US companies → SOC 2
  └── International → ISO 27001
  └── Both → SOC 2 + ISO 27001

Running containers?
  └── Add CIS Benchmarks for technical controls
```

---

## Unified Compliance Mapping

### The "Comply Once, Satisfy Many" Approach

Instead of implementing controls per-framework, implement the **strictest requirement** and satisfy all frameworks simultaneously.

### Control Mapping Matrix

#### Access Control

| Control | SOC 2 | HIPAA | PCI DSS | GDPR | ISO 27001 | Strictest |
|---------|-------|-------|---------|------|-----------|-----------|
| Unique user IDs | CC6.1 | §164.312(a)(2)(i) | 8.1.1 | Art 32 | A.9.2.1 | All equal |
| MFA | CC6.1 | Addressable | 8.3.1 | Art 32 | A.9.4.2 | **PCI DSS** (required) |
| Password complexity | CC6.1 | Addressable | 8.3.6 | - | A.9.4.3 | **PCI DSS** (12+ chars) |
| Session timeout | CC6.1 | §164.312(a)(2)(iii) | 8.1.8 | - | A.11.2.8 | **PCI DSS** (15 min) |
| Access logging | CC7.2 | §164.312(b) | 10.2 | Art 30 | A.12.4.1 | **HIPAA** (6yr retention) |

**Unified Implementation:**
```typescript
// Satisfies: SOC 2, HIPAA, PCI DSS, GDPR, ISO 27001
const accessControlConfig = {
  authentication: {
    mfaRequired: true,              // PCI DSS 8.3.1
    passwordMinLength: 12,          // PCI DSS 8.3.6
    passwordRequirements: {
      uppercase: true,
      lowercase: true,
      numbers: true,
      special: true
    },
    maxFailedAttempts: 6,           // PCI DSS 8.1.6
    lockoutDuration: 30 * 60 * 1000 // 30 minutes
  },
  session: {
    maxAge: 15 * 60 * 1000,         // 15 min (PCI DSS 8.1.8)
    secure: true,
    httpOnly: true,
    sameSite: 'strict'
  },
  logging: {
    retentionYears: 6,              // HIPAA maximum
    events: ['auth', 'access', 'admin', 'data']
  }
};
```

#### Encryption

| Control | SOC 2 | HIPAA | PCI DSS | GDPR | ISO 27001 | Strictest |
|---------|-------|-------|---------|------|-----------|-----------|
| TLS version | CC6.7 | §164.312(e)(1) | 4.1 | Art 32 | A.13.1.1 | **PCI DSS** (TLS 1.2+) |
| Cipher suites | - | - | Appendix A2 | - | - | **PCI DSS** (specific list) |
| Data at rest | CC6.7 | §164.312(a)(2)(iv) | 3.4 | Art 32 | A.10.1.1 | **PCI DSS** (AES-256) |
| Key management | CC6.1 | Addressable | 3.5-3.6 | Art 32 | A.10.1.2 | **PCI DSS** (rotation) |

**Unified Implementation:**
```nginx
# nginx.conf - Satisfies all frameworks
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305;
ssl_prefer_server_ciphers on;
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_stapling on;
ssl_stapling_verify on;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

#### Audit Logging

| Control | SOC 2 | HIPAA | PCI DSS | GDPR | ISO 27001 | Strictest |
|---------|-------|-------|---------|------|-----------|-----------|
| Retention period | 1 year | 6 years | 1 year | Varies | Defined | **HIPAA** (6 years) |
| Tamper protection | CC7.2 | §164.312(c)(1) | 10.5 | Art 32 | A.12.4.2 | All equal |
| Time sync | CC7.2 | - | 10.4 | - | A.12.4.4 | **PCI DSS** (NTP) |
| Log review | CC7.2 | §164.308(a)(1)(ii)(D) | 10.6 | - | A.12.4.1 | **PCI DSS** (daily) |

**Unified Implementation:**
```typescript
// Audit log structure - Satisfies all frameworks
interface AuditLog {
  // Required by all
  timestamp: string;        // ISO 8601, NTP synced (PCI 10.4)
  eventType: string;
  userId: string;
  outcome: 'SUCCESS' | 'FAILURE';

  // Access details (PCI 10.2, HIPAA)
  ipAddress: string;
  userAgent: string;

  // Resource details (GDPR Art 30)
  resource: string;
  resourceId: string;
  action: 'CREATE' | 'READ' | 'UPDATE' | 'DELETE';

  // Data classification (GDPR, HIPAA)
  dataClassification: 'PUBLIC' | 'INTERNAL' | 'CONFIDENTIAL' | 'PHI' | 'PCI';

  // Immutability hash (PCI 10.5, SOC 2)
  previousHash?: string;
  hash: string;
}

// Retention: 6 years (HIPAA maximum)
// Review: Daily (PCI DSS requirement)
// Storage: Immutable, encrypted (All frameworks)
```

#### Vulnerability Management

| Control | SOC 2 | HIPAA | PCI DSS | GDPR | ISO 27001 | Strictest |
|---------|-------|-------|---------|------|-----------|-----------|
| Scan frequency | Periodic | Risk-based | Quarterly | - | A.12.6.1 | **PCI DSS** (quarterly ASV) |
| Patch timeline | Reasonable | Reasonable | 30 days critical | - | A.12.6.1 | **PCI DSS** (30 days) |
| Pen testing | Annual | Recommended | Annual + after changes | - | A.18.2.3 | **PCI DSS** (+ segmentation) |

**Unified Implementation:**
```yaml
# CI/CD Pipeline - Satisfies all frameworks
vulnerability_management:
  container_scanning:
    tool: trivy
    frequency: every_build
    fail_on: HIGH,CRITICAL

  dependency_scanning:
    tool: npm_audit
    frequency: daily
    fail_on: high

  asv_scanning:  # PCI DSS requirement
    frequency: quarterly
    vendor: approved_scanning_vendor

  penetration_testing:
    frequency: annual
    after_significant_changes: true
    scope: full_application

  patching:
    critical: 72_hours    # Stricter than PCI (30 days)
    high: 7_days
    medium: 30_days
    low: 90_days
```

---

## Framework-Specific Requirements

### PCI DSS v4.0 (Payment Card Data)

**When Required:** Any storage, processing, or transmission of cardholder data

| Requirement | Description | Implementation |
|-------------|-------------|----------------|
| **Req 3** | Protect stored cardholder data | Encryption, tokenization, truncation |
| **Req 4** | Encrypt transmission | TLS 1.2+ only |
| **Req 6** | Develop secure systems | Secure SDLC, code review |
| **Req 8** | Identify and authenticate | MFA, strong passwords |
| **Req 10** | Log and monitor | Audit trails, daily review |
| **Req 11** | Test security | Quarterly scans, annual pen test |
| **Req 12** | Maintain policy | Security policies, training |

**Critical Technical Controls:**
```typescript
// PCI DSS Cardholder Data Handling
// NEVER store full PAN, CVV, or PIN

// ✅ Tokenization (Recommended)
const token = await paymentGateway.tokenize(cardNumber);
await db.save({ userId, paymentToken: token });

// ✅ Truncation for display
const maskedPan = `****-****-****-${cardNumber.slice(-4)}`;

// ❌ NEVER DO THIS
// await db.save({ cardNumber, cvv, expiryDate });
```

### GDPR (EU Personal Data)

**When Required:** Processing personal data of EU residents

| Article | Requirement | Implementation |
|---------|-------------|----------------|
| **Art 5** | Data minimization | Collect only necessary data |
| **Art 6** | Lawful basis | Document legal basis for processing |
| **Art 7** | Consent | Explicit, withdrawable consent |
| **Art 15-20** | Data subject rights | Export, delete, portability |
| **Art 25** | Privacy by design | Default privacy settings |
| **Art 32** | Security | Encryption, access control |
| **Art 33** | Breach notification | 72 hours to authority |

**Data Subject Rights Implementation:**
```typescript
// GDPR Data Subject Rights API

// Art 15 - Right of Access
app.get('/api/gdpr/my-data', authenticate, async (req, res) => {
  const userData = await exportUserData(req.user.id);
  res.json(userData);
});

// Art 17 - Right to Erasure
app.delete('/api/gdpr/my-data', authenticate, async (req, res) => {
  // Check for legal holds (HIPAA may override!)
  if (await hasLegalHold(req.user.id)) {
    return res.status(403).json({
      error: 'Data subject to legal retention requirement'
    });
  }
  await anonymizeUserData(req.user.id);
  res.json({ status: 'Data anonymized' });
});

// Art 20 - Right to Portability
app.get('/api/gdpr/export', authenticate, async (req, res) => {
  const data = await exportUserData(req.user.id);
  res.setHeader('Content-Type', 'application/json');
  res.setHeader('Content-Disposition', 'attachment; filename=my-data.json');
  res.json(data);
});
```

### ISO 27001 (Information Security Management)

**When Required:** International enterprise sales, certification requirement

| Control | Description | Implementation |
|---------|-------------|----------------|
| **A.5** | Information security policies | Documented, reviewed annually |
| **A.6** | Organization of security | Roles, responsibilities, segregation |
| **A.8** | Asset management | Inventory, classification, handling |
| **A.9** | Access control | Policy, user management, system controls |
| **A.10** | Cryptography | Key management, encryption policy |
| **A.12** | Operations security | Procedures, malware, backup, logging |
| **A.14** | System development | Secure SDLC, testing, data protection |

**Key Difference from SOC 2:** ISO 27001 requires a formal Information Security Management System (ISMS) with continuous improvement (Plan-Do-Check-Act).

---

## Conflict Resolution

### GDPR vs HIPAA Conflicts

| Scenario | GDPR Says | HIPAA Says | Resolution |
|----------|-----------|------------|------------|
| Right to erasure | Must delete on request | Retain PHI 6 years | **HIPAA wins** (legal requirement) |
| Breach notification | 72 hours to authority | 60 days to individuals | **Do both** (different recipients) |
| Consent | Explicit required | Implied for treatment | **Get explicit** (satisfies both) |
| Data portability | Required | No requirement | **Implement** (satisfies GDPR) |

**Implementation Pattern:**
```typescript
async function handleDeletionRequest(userId: string): Promise<DeletionResult> {
  const user = await getUser(userId);

  // Check for HIPAA retention requirements
  const phiRecords = await getPHIRecords(userId);
  const retentionEnd = phiRecords.map(r =>
    addYears(r.lastModified, 6)
  );

  const earliestDeletion = max(retentionEnd);

  if (earliestDeletion > new Date()) {
    // Cannot delete - HIPAA retention applies
    return {
      status: 'RETAINED',
      reason: 'HIPAA_RETENTION',
      eligibleForDeletion: earliestDeletion,
      // GDPR requires we inform the user
      message: `Data subject to healthcare retention until ${earliestDeletion.toISOString()}`
    };
  }

  // Safe to delete/anonymize
  await anonymizeUserData(userId);
  return { status: 'DELETED' };
}
```

### PCI DSS vs Other Frameworks

PCI DSS is generally the **most prescriptive**. When in doubt:

1. Follow PCI DSS technical requirements (they're specific)
2. Add HIPAA retention periods (6 years vs PCI's 1 year)
3. Implement GDPR data subject rights
4. Document for SOC 2/ISO 27001 audit

---

## Compliance Framework Overview

### SOC 2 Type II Trust Service Criteria

| Criteria | Description | Relevant Sections |
|----------|-------------|-------------------|
| **Security** | Protection against unauthorized access | Access Control, Encryption, Container Security |
| **Availability** | System uptime and performance | Health Checks, Monitoring |
| **Processing Integrity** | Accurate and timely processing | Audit Logging, Data Validation |
| **Confidentiality** | Protection of confidential data | Encryption, Secrets Management |
| **Privacy** | Personal data handling | Data Classification, Access Controls |

### HIPAA Technical Safeguards (45 CFR § 164.312)

| Requirement | Description | Implementation |
|-------------|-------------|----------------|
| **Access Control** (a)(1) | Unique user identification | Authentication, RBAC |
| **Audit Controls** (b) | Record and examine activity | Logging, Monitoring |
| **Integrity** (c)(1) | Protect ePHI from alteration | Checksums, Immutable logs |
| **Transmission Security** (e)(1) | Encrypt ePHI in transit | TLS, HTTPS |
| **Encryption** (a)(2)(iv) | Encrypt ePHI at rest | Database encryption |

---

## Container Security Requirements

### 1. Run as Non-Root User

**SOC 2:** Security control — principle of least privilege
**HIPAA:** Access control — minimum necessary access

```dockerfile
# ❌ Bad - runs as root
FROM node:20-alpine
COPY . .
CMD ["node", "server.js"]

# ✅ Good - runs as non-root user
FROM node:20-alpine

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 -G nodejs

WORKDIR /app
COPY --chown=nodejs:nodejs . .

# Switch to non-root user
USER nodejs

CMD ["node", "server.js"]
```

### 2. Use Read-Only File System

**SOC 2:** Integrity control — prevent unauthorized modifications

```yaml
# docker-compose.yml
services:
  api:
    image: myapp:latest
    read_only: true
    tmpfs:
      - /tmp
      - /app/logs  # If logs need to be written
    security_opt:
      - no-new-privileges:true
```

### 3. Limit Container Capabilities

**SOC 2:** Security control — minimize attack surface

```yaml
# docker-compose.yml
services:
  api:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # Only if binding to ports < 1024
```

### 4. Scan Images for Vulnerabilities

**SOC 2:** Security control — vulnerability management
**HIPAA:** Risk analysis requirement

```bash
# Using Trivy (recommended)
trivy image --severity HIGH,CRITICAL myapp:latest

# Using Docker Scout
docker scout cve myapp:latest

# Using Snyk
snyk container test myapp:latest
```

**CI/CD Integration:**
```yaml
# GitHub Actions example
- name: Scan image for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.IMAGE_NAME }}
    exit-code: '1'
    severity: 'CRITICAL,HIGH'
```

---

## Secrets Management

### Never Commit Secrets

**SOC 2:** Confidentiality control
**HIPAA:** Access control — authentication credentials

```bash
# .gitignore - MUST include:
.env
.env.*
*.pem
*.key
credentials.json
secrets/
```

### Use Environment Variables (Not Files in Images)

```dockerfile
# ❌ Bad - secrets in image
COPY .env /app/.env

# ✅ Good - secrets injected at runtime
# (No COPY of secrets; inject via docker-compose or orchestrator)
```

```yaml
# docker-compose.yml
services:
  api:
    environment:
      - DATABASE_URL  # Loaded from host environment
    env_file:
      - .env  # File not included in image
```

### Secrets Rotation

**SOC 2:** Periodic credential rotation requirement

```bash
# Generate strong random passwords
openssl rand -base64 32

# In install.sh - generate unique credentials per installation
DB_PASSWORD=$(openssl rand -base64 32 | tr -d '/+=' | head -c 32)
JWT_SECRET=$(openssl rand -base64 64 | tr -d '/+=' | head -c 64)
```

### Use Secrets Managers in Production

| Tool | Use Case | Reference |
|------|----------|-----------|
| Docker Secrets | Docker Swarm | [Docker Docs](https://docs.docker.com/engine/swarm/secrets/) |
| Kubernetes Secrets | K8s deployments | [K8s Docs](https://kubernetes.io/docs/concepts/configuration/secret/) |
| AWS Secrets Manager | AWS deployments | [AWS Docs](https://aws.amazon.com/secrets-manager/) |
| HashiCorp Vault | Multi-cloud | [Vault Docs](https://www.vaultproject.io/docs) |

---

## Encryption Requirements

### Encryption in Transit (TLS)

**HIPAA:** Transmission security — ePHI encryption
**SOC 2:** Confidentiality control

```yaml
# docker-compose.yml with TLS termination
services:
  nginx:
    image: nginx:alpine
    volumes:
      - ./certs:/etc/nginx/certs:ro
    ports:
      - "443:443"
    environment:
      - SSL_CERTIFICATE=/etc/nginx/certs/cert.pem
      - SSL_CERTIFICATE_KEY=/etc/nginx/certs/key.pem
```

**Minimum TLS version: 1.2** (TLS 1.3 preferred)

```nginx
# nginx.conf
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
```

### Encryption at Rest

**HIPAA:** Encryption addressable requirement
**SOC 2:** Confidentiality control

```yaml
# PostgreSQL with encryption
services:
  postgres:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      # Enable SSL for client connections
      POSTGRES_INITDB_ARGS: "--data-checksums"
    command: >
      postgres
      -c ssl=on
      -c ssl_cert_file=/var/lib/postgresql/server.crt
      -c ssl_key_file=/var/lib/postgresql/server.key
```

**Database Connection String:**
```
DATABASE_URL="postgresql://user:pass@host:5432/db?sslmode=require"
```

---

## Audit Logging

### What to Log (HIPAA Requirement)

**HIPAA:** Audit controls — record and examine access

| Event Type | Data to Capture | Retention |
|------------|-----------------|-----------|
| Authentication | User ID, timestamp, IP, success/fail | 6 years (HIPAA) |
| Authorization | Resource accessed, action, user | 6 years |
| Data Access | PHI accessed, by whom, when | 6 years |
| Configuration Changes | What changed, by whom, when | 6 years |
| System Events | Startup, shutdown, errors | 1 year |

### Structured Logging Format

```typescript
// Compliant log format
const auditLog = {
  timestamp: new Date().toISOString(),
  eventType: 'DATA_ACCESS',
  userId: user.id,
  userEmail: user.email,  // For identification
  action: 'READ',
  resource: 'patient_records',
  resourceId: recordId,
  ipAddress: request.ip,
  userAgent: request.headers['user-agent'],
  outcome: 'SUCCESS',
  // Never log PHI/PII in plain text!
};

logger.info(JSON.stringify(auditLog));
```

### Log Shipping & Retention

```yaml
# docker-compose.yml with log shipping
services:
  api:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"
        labels: "service,environment"
    labels:
      - "service=api"
      - "environment=production"
```

**External log aggregation:** Consider Elasticsearch, Splunk, or CloudWatch for compliance retention.

---

## Access Control

### Role-Based Access Control (RBAC)

**HIPAA:** Access control — minimum necessary
**SOC 2:** Security — authorization controls

```typescript
// Prisma schema example
model User {
  id        String   @id @default(uuid())
  email     String   @unique
  role      Role     @default(USER)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

enum Role {
  USER
  ADMIN
  AUDITOR  // Read-only access to logs
}

// Middleware for role checking
const requireRole = (roles: Role[]) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      // Log unauthorized access attempt
      auditLogger.warn({
        eventType: 'AUTHORIZATION_FAILURE',
        userId: req.user.id,
        attemptedResource: req.path,
        requiredRoles: roles,
      });
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
};
```

### Session Management

**SOC 2:** Access control — session timeout

```typescript
// Session configuration
const sessionConfig = {
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: true,          // HTTPS only
    httpOnly: true,        // No JavaScript access
    sameSite: 'strict',    // CSRF protection
    maxAge: 15 * 60 * 1000 // 15 minutes (HIPAA recommendation)
  }
};
```

---

## Environment Separation

### Separate Production from Development

**SOC 2:** Change management — environment isolation

| Environment | Data | Access | Logging |
|-------------|------|--------|---------|
| Development | Synthetic/test only | Developers | Minimal |
| Staging | Anonymized production copy | QA + Developers | Standard |
| Production | Real data | Limited access | Full audit |

**Never use production data in development** (HIPAA violation risk)

```bash
# Environment-specific .env files
.env.development    # Local dev settings
.env.staging        # Staging settings
.env.production     # Production (never commit!)
```

### Network Segmentation

```yaml
# docker-compose.yml with isolated networks
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No external access

services:
  nginx:
    networks:
      - frontend
      - backend

  api:
    networks:
      - backend  # Only accessible via nginx

  postgres:
    networks:
      - backend  # Only accessible by api
```

---

## Compliance Checklist

### Pre-Deployment (SOC 2 + HIPAA)

#### Container Security
- [ ] Containers run as non-root user
- [ ] Read-only file systems where possible
- [ ] Capabilities dropped (except required)
- [ ] No secrets in image layers
- [ ] Base images from trusted sources
- [ ] Images scanned for vulnerabilities (HIGH/CRITICAL = 0)
- [ ] Image signing enabled (optional but recommended)

#### Encryption
- [ ] TLS 1.2+ for all network communication
- [ ] Database connections use SSL
- [ ] Secrets encrypted at rest
- [ ] No hardcoded credentials

#### Access Control
- [ ] RBAC implemented
- [ ] Session timeouts configured (≤15 min for PHI)
- [ ] Failed login attempts logged
- [ ] Account lockout after failed attempts

#### Audit Logging
- [ ] Authentication events logged
- [ ] Authorization failures logged
- [ ] Data access logged (who, what, when)
- [ ] Logs shipped to secure, immutable storage
- [ ] Log retention meets requirements (6 years HIPAA)

#### Network Security
- [ ] Internal services not exposed publicly
- [ ] Network segmentation implemented
- [ ] Firewall rules documented
- [ ] API rate limiting enabled

### Ongoing Compliance

| Task | Frequency | SOC 2 | HIPAA |
|------|-----------|-------|-------|
| Vulnerability scans | Weekly | ✅ | ✅ |
| Dependency updates | Monthly | ✅ | ✅ |
| Access reviews | Quarterly | ✅ | ✅ |
| Penetration testing | Annual | ✅ | Recommended |
| Risk assessment | Annual | ✅ | ✅ |
| Policy reviews | Annual | ✅ | ✅ |
| Backup testing | Quarterly | ✅ | ✅ |

---

## Incident Response

### Required Documentation

**HIPAA:** Breach notification requirements

```markdown
## Incident Report Template

**Incident ID:** INC-2025-001
**Date Detected:** YYYY-MM-DD HH:MM UTC
**Date Contained:** YYYY-MM-DD HH:MM UTC

### Summary
Brief description of the incident.

### Impact Assessment
- [ ] PHI potentially accessed
- [ ] Number of records affected: ___
- [ ] Types of data exposed: ___

### Root Cause
Technical description of what failed.

### Remediation Steps
1. Immediate containment actions
2. Long-term fixes implemented
3. Prevention measures added

### Notifications Required
- [ ] HHS (if >500 individuals affected)
- [ ] Affected individuals (within 60 days)
- [ ] State attorneys general
- [ ] Media (if >500 in single state)
```

### Breach Notification Timeline (HIPAA)

| Condition | Notification Deadline |
|-----------|----------------------|
| Breach affecting <500 individuals | Within 60 days of discovery |
| Breach affecting ≥500 individuals | Within 60 days + notify HHS immediately + media |
| Business Associate breach | Notify Covered Entity within 60 days |

---

## References & Resources

### Compliance Frameworks
- [SOC 2 Trust Service Criteria](https://www.aicpa.org/resources/article/soc-2-trust-services-criteria) — AICPA official guide
- [HIPAA Security Rule](https://www.hhs.gov/hipaa/for-professionals/security/index.html) — HHS official guidance
- [HIPAA Technical Safeguards](https://www.hhs.gov/hipaa/for-professionals/security/guidance/index.html) — Implementation specs
- [PCI DSS v4.0](https://www.pcisecuritystandards.org/document_library/) — Payment Card Industry standards
- [PCI DSS Quick Reference](https://www.pcisecuritystandards.org/pdfs/pci_ssc_quick_guide.pdf) — Summary guide
- [GDPR Official Text](https://gdpr-info.eu/) — Full regulation with commentary
- [GDPR Compliance Checklist](https://gdpr.eu/checklist/) — Implementation guide
- [ISO 27001 Overview](https://www.iso.org/isoiec-27001-information-security.html) — Official ISO page
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework) — Risk management framework
- [NIST 800-53 Control Catalog](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final) — Comprehensive controls

### Framework Mapping Resources
- [NIST CSF to ISO 27001 Mapping](https://www.nist.gov/cyberframework/framework) — Official crosswalk
- [Unified Compliance Framework](https://www.unifiedcompliance.com/) — Control mapping database
- [HITRUST CSF](https://hitrustalliance.net/csf/) — Healthcare framework (maps to HIPAA, SOC 2, NIST)

### Container Security
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker) — Security configuration
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes) — K8s security
- [NIST SP 800-190](https://csrc.nist.gov/publications/detail/sp/800-190/final) — Container security guide
- [OWASP Docker Security](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html) — Security checklist

### Tools
- [Trivy](https://aquasecurity.github.io/trivy/) — Vulnerability scanner
- [Falco](https://falco.org/) — Runtime security monitoring
- [OPA/Gatekeeper](https://www.openpolicyagent.org/) — Policy enforcement
- [Vault](https://www.vaultproject.io/) — Secrets management
- [Snyk](https://snyk.io/) — Dependency and container scanning

### Audit & Logging
- [ELK Stack](https://www.elastic.co/elastic-stack) — Log aggregation
- [Splunk](https://www.splunk.com/) — Enterprise logging
- [AWS CloudTrail](https://aws.amazon.com/cloudtrail/) — AWS audit logging
- [Datadog](https://www.datadoghq.com/) — Observability and compliance

### Compliance Platforms
- [Vanta](https://www.vanta.com/) — SOC 2, HIPAA, ISO 27001, GDPR automation
- [Drata](https://drata.com/) — Continuous compliance monitoring
- [Secureframe](https://secureframe.com/) — SOC 2, HIPAA, PCI DSS, ISO 27001
- [Sprinto](https://sprinto.com/) — Compliance automation
- [Tugboat Logic](https://tugboatlogic.com/) — Security assurance platform

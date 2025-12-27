# Security & Compliance Guide

> **Scope:** This guide covers security controls and compliance requirements for containerized Node.js applications, with specific focus on **SOC 2 Type II** and **HIPAA** requirements.

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
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework) — Risk management framework

### Container Security
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker) — Security configuration
- [NIST SP 800-190](https://csrc.nist.gov/publications/detail/sp/800-190/final) — Container security guide
- [OWASP Docker Security](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html) — Security checklist

### Tools
- [Trivy](https://aquasecurity.github.io/trivy/) — Vulnerability scanner
- [Falco](https://falco.org/) — Runtime security monitoring
- [OPA/Gatekeeper](https://www.openpolicyagent.org/) — Policy enforcement
- [Vault](https://www.vaultproject.io/) — Secrets management

### Audit & Logging
- [ELK Stack](https://www.elastic.co/elastic-stack) — Log aggregation
- [Splunk](https://www.splunk.com/) — Enterprise logging
- [AWS CloudTrail](https://aws.amazon.com/cloudtrail/) — AWS audit logging

### Compliance Platforms
- [Vanta](https://www.vanta.com/) — Automated SOC 2 compliance
- [Drata](https://drata.com/) — Continuous compliance monitoring
- [Secureframe](https://secureframe.com/) — SOC 2 + HIPAA automation

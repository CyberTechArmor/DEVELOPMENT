# Full-Stack Development Best Practices & Common Issues

> A comprehensive reference guide for Node.js, TypeScript, React, Prisma, Docker, and compliance-focused development.

---

## Table of Contents

1. [Technology Stack Overview](#technology-stack-overview)
2. [Prisma & Database](#prisma--database)
3. [Docker & Containerization](#docker--containerization)
4. [TypeScript & Build](#typescript--build)
5. [Backend (Fastify/Node.js)](#backend-fastifynodejs)
6. [Frontend (React/Vite)](#frontend-reactvite)
7. [Security & Secrets](#security--secrets)
8. [Compliance Quick Reference](#compliance-quick-reference)
9. [Production Checklist](#production-checklist)
10. [Quick Command Reference](#quick-command-reference)

---

## Technology Stack Overview

### Recommended Stack

| Layer | Technology | Version |
|-------|------------|---------|
| Runtime | Node.js LTS | 20+ |
| Language | TypeScript | 5.3+ |
| Backend | Fastify | 4.25+ |
| Database | PostgreSQL | 15+ |
| ORM | Prisma | 5.22+ |
| Cache/Queue | Redis + BullMQ | 7+ / 5.x |
| Frontend | React + Vite | 18.2+ / 5.x |
| State | Zustand + React Query | 4.4+ / 5.x |
| Styling | TailwindCSS | 3.4+ |
| Reverse Proxy | nginx | alpine |
| Containerization | Docker | latest |

### Architecture Diagram

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Browser   │────▶│    nginx    │────▶│   Fastify   │
│  (React)    │     │  (TLS/Proxy)│     │  (API)      │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
                    ┌──────────────────────────┼──────────────────────────┐
                    │                          │                          │
              ┌─────▼─────┐             ┌──────▼──────┐            ┌──────▼──────┐
              │ PostgreSQL│             │    Redis    │            │   LiveKit   │
              │    (DB)   │             │(Cache/Queue)│            │  (WebRTC)   │
              └───────────┘             └─────────────┘            └─────────────┘
```

---

## Prisma & Database

### Core Concepts: Know the Difference

| Command | What It Does | When to Use |
|---------|--------------|-------------|
| `prisma generate` | Creates TypeScript client from schema | After any schema change |
| `prisma migrate dev` | Creates migration files + applies them | Development only |
| `prisma migrate deploy` | Applies existing migration files | Production deployments |
| `prisma db push` | Pushes schema directly (no migrations) | Prototyping, Docker-only setups |
| `prisma db seed` | Runs seed.ts to populate data | After fresh database setup |

### Common Issues & Solutions

#### Issue: Types not updating after schema changes

```bash
# Problem: Added new model but TypeScript doesn't recognize it
# Solution: Regenerate the Prisma client
npx prisma generate
```

#### Issue: "Model X does not exist" in production

```bash
# Problem: Schema was pushed but migrations weren't created
# Solution: In production, always use migrations
npx prisma migrate deploy  # NOT db push
```

#### Issue: Seed script fails in production

```typescript
// Problem: tsx/ts-node not available in production container
// Solution 1: Compile seed to JavaScript
// In package.json scripts:
"prisma:seed:build": "tsc prisma/seed.ts --outDir prisma/dist --esModuleInterop --skipLibCheck",
"prisma:seed:prod": "node prisma/dist/seed.js"

// Solution 2: Use tsx in production (add to dependencies, not devDependencies)
"dependencies": {
  "tsx": "^4.x"
}
```

#### Issue: Type inference with related models

```typescript
// DON'T: Import types that might not exist
import { User } from '@prisma/client';  // Basic type only

// DO: Use Prisma's type inference for relations
import { Prisma } from '@prisma/client';

type UserWithPosts = Prisma.UserGetPayload<{
  include: { posts: true }
}>;
```

#### Issue: `.prisma/client` not found in monorepo

```dockerfile
# Problem: Prisma generates to wrong location in monorepos
# Solution: Specify output in schema.prisma
generator client {
  provider = "prisma-client-js"
  output   = "../node_modules/.prisma/client"
}

# Or copy it explicitly in Dockerfile
COPY --from=builder /app/node_modules/.prisma /app/node_modules/.prisma
```

### Best Practices

1. **Always regenerate after schema changes** - Add to pre-commit hooks
2. **Use migrations for production** - `db push` is for prototyping only
3. **Version pin Prisma** - Use exact versions (`5.22.0` not `^5.22.0`)
4. **Include in Docker builds** - Run `prisma generate` before `npm run build`
5. **Validate schema in CI** - Run `prisma validate` in your pipeline

### Pre-commit Hook for Prisma

```bash
# .husky/pre-commit
npx prisma generate
npx prisma validate
git add prisma/
npx tsc --noEmit
```

---

## Docker & Containerization

### The #1 Docker Issue: npx Downloads Latest Versions

```dockerfile
# WRONG: npx ignores your pinned versions and downloads latest
RUN npx prisma generate  # Downloads latest prisma, not your pinned version!

# CORRECT: Use the locally installed version
RUN ./node_modules/.bin/prisma generate

# OR: Use npm script that references local binary
RUN npm run prisma:generate
```

**Why this matters**: Your `package.json` pins `prisma@5.22.0`, but `npx prisma` downloads and runs the latest version (e.g., `5.25.0`), causing version mismatches and unexpected behavior.

### Common Issues & Solutions

#### Issue: Silent failures with `|| true`

```dockerfile
# DANGEROUS: Hides compilation errors
RUN npm run build || true  # Build fails silently, app crashes at runtime

# CORRECT: Let builds fail loudly
RUN npm run build
# If you need conditional logic, be explicit:
RUN npm run build || (echo "Build failed" && exit 1)
```

#### Issue: Prisma fails on Alpine - missing OpenSSL

```dockerfile
# Problem: Prisma requires OpenSSL which isn't in Alpine by default
# Solution: Install it
FROM node:20-alpine
RUN apk add --no-cache openssl
```

#### Issue: devDependencies missing in production

```dockerfile
# Problem: CLI tools in devDependencies aren't available after npm ci --omit=dev

# Solution 1: Move runtime CLIs to dependencies
"dependencies": {
  "prisma": "5.22.0",  # Needed for migrations
  "tsx": "4.x"         # Needed for seed scripts
}

# Solution 2: Multi-stage build - generate before pruning
FROM node:20-alpine AS builder
COPY package*.json ./
RUN npm ci
RUN ./node_modules/.bin/prisma generate
RUN npm run build

FROM node:20-alpine AS runner
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma
RUN npm ci --omit=dev
```

#### Issue: Monorepo packages not found

```dockerfile
# Problem: npm workspaces hoist differently than expected

# Solution: Set NODE_PATH and copy workspace packages
ENV NODE_PATH=/app/node_modules:/app/packages/shared/node_modules
COPY packages/shared/package*.json ./packages/shared/
COPY packages/api/package*.json ./packages/api/
RUN npm ci --workspace=packages/api --include-workspace-root
```

#### Issue: Low memory on VPS causing crashes

```dockerfile
# Solution: Configure Node.js memory limits
ENV NODE_OPTIONS="--max-old-space-size=512"

# docker-compose.yml
services:
  api:
    deploy:
      resources:
        limits:
          memory: 1G
        reservations:
          memory: 512M
```

### Multi-Stage Build Template

```dockerfile
# Stage 1: Dependencies
FROM node:20-alpine AS deps
WORKDIR /app
RUN apk add --no-cache openssl python3 make g++
COPY package*.json ./
RUN npm ci

# Stage 2: Builder
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN ./node_modules/.bin/prisma generate
RUN npm run build

# Stage 3: Runner (Production)
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

RUN apk add --no-cache openssl
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 appuser

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma
COPY --from=builder /app/package*.json ./

RUN npm ci --omit=dev

USER appuser
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Best Practices

1. **Never use `npx` in Dockerfiles** - Use `./node_modules/.bin/` or npm scripts
2. **Never use `|| true`** - Let builds fail loudly
3. **Use multi-stage builds** - Separate build and runtime stages
4. **Pin exact versions** - `"prisma": "5.22.0"` not `"^5.22.0"`
5. **Install Alpine dependencies** - OpenSSL for Prisma, python3/make/g++ for native modules
6. **Run as non-root** - Create and use a dedicated user
7. **Set NODE_ENV=production** - Enables optimizations
8. **Use `npm ci`** - Faster and more reliable than `npm install`

---

## TypeScript & Build

### Common Issues & Solutions

#### Issue: Standalone tsc can't resolve path aliases

```json
// Problem: TypeScript paths don't work after tsc compilation
// tsconfig.json
{
  "compilerOptions": {
    "paths": { "@/*": ["./src/*"] }  // These aren't resolved by tsc alone
  }
}

// Solution: Use a bundler (tsup, esbuild) or path resolver
// package.json
"scripts": {
  "build": "tsup src/index.ts --format cjs,esm --dts"
}
```

#### Issue: Type errors not caught before deploy

```json
// Solution: Add type checking to CI and pre-commit
// package.json
"scripts": {
  "typecheck": "tsc --noEmit",
  "lint": "eslint . && npm run typecheck"
}

// .husky/pre-commit
npm run typecheck
```

#### Issue: ESM vs CommonJS conflicts

```json
// Solution: Be explicit about module system
// package.json
{
  "type": "module",  // Use ESM
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    }
  }
}

// tsconfig.json
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler"
  }
}
```

### Best Practices

1. **Enable strict mode** - `"strict": true` in tsconfig.json
2. **Run typecheck in CI** - Catch errors before deployment
3. **Use a bundler for production** - tsup, esbuild, or rollup
4. **Keep TypeScript updated** - New versions include important fixes
5. **Use project references for monorepos** - Better incremental builds

---

## Backend (Fastify/Node.js)

### Security Setup Template

```typescript
import Fastify from 'fastify';
import helmet from '@fastify/helmet';
import rateLimit from '@fastify/rate-limit';
import cors from '@fastify/cors';

const app = Fastify({
  logger: {
    level: process.env.LOG_LEVEL || 'info',
    redact: ['req.headers.authorization', 'req.body.password', 'req.body.ssn']
  }
});

// Security headers
await app.register(helmet, {
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
    }
  }
});

// Rate limiting
await app.register(rateLimit, {
  max: 100,
  timeWindow: '1 minute'
});

// CORS
await app.register(cors, {
  origin: process.env.ALLOWED_ORIGINS?.split(',') || false,
  credentials: true
});
```

### Common Issues & Solutions

#### Issue: Fastify 4.x plugin registration changed

```typescript
// Fastify 4.x requires await for plugin registration
// WRONG (Fastify 3.x style):
app.register(cors, { origin: true });

// CORRECT (Fastify 4.x):
await app.register(cors, { origin: true });
```

#### Issue: DNS resolution delays with external services

```typescript
// Solution: Cache DNS lookups
import { lookup } from 'dns';
import { Agent } from 'http';

const agent = new Agent({
  keepAlive: true,
  lookup: (hostname, options, callback) => {
    // Use your caching DNS resolver
    lookup(hostname, options, callback);
  }
});
```

### nginx Configuration for Node.js

```nginx
upstream api {
    server api:3000;
    keepalive 32;
}

# DNS caching for external services
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;

server {
    listen 443 ssl http2;
    server_name api.example.com;

    # TLS 1.2+ only (compliance requirement)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options DENY always;

    location /api/ {
        proxy_pass http://api;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### Best Practices

1. **Always use Helmet** - Security headers are essential
2. **Implement rate limiting** - Prevent abuse
3. **Redact sensitive data in logs** - Passwords, tokens, PII
4. **Use structured logging (Pino)** - JSON format for log aggregation
5. **Set appropriate timeouts** - Prevent hanging connections
6. **Use connection pooling** - For database and Redis

---

## Frontend (React/Vite)

### State Management Decision

| Use Zustand for | Use React Query for |
|-----------------|---------------------|
| UI state (modals, themes) | Server data (API responses) |
| Form state | Cached data |
| Client-only state | Data that needs refetching |

### Common Issues & Solutions

#### Issue: Authentication state not syncing across tabs

```typescript
// Solution: Use storage events for cross-tab sync
// authStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

export const useAuthStore = create(
  persist(
    (set) => ({
      user: null,
      setUser: (user) => set({ user }),
      logout: () => set({ user: null }),
    }),
    {
      name: 'auth-storage',
      // Listen for changes from other tabs
      onRehydrateStorage: () => (state) => {
        window.addEventListener('storage', (e) => {
          if (e.key === 'auth-storage') {
            const newState = JSON.parse(e.newValue || '{}');
            if (newState.state?.user !== state?.user) {
              window.location.reload();
            }
          }
        });
      },
    }
  )
);
```

#### Issue: JWT expiration not handled gracefully

```typescript
// Solution: Implement token refresh interceptor
import axios from 'axios';

const api = axios.create({ baseURL: '/api' });

api.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401 && !error.config._retry) {
      error.config._retry = true;
      try {
        await refreshToken();
        return api(error.config);
      } catch {
        // Refresh failed, redirect to login
        window.location.href = '/login';
      }
    }
    return Promise.reject(error);
  }
);
```

### Best Practices

1. **Separate server and client state** - React Query for server, Zustand for client
2. **Use React Query's stale-while-revalidate** - Better UX with background updates
3. **Implement error boundaries** - Graceful error handling
4. **Lazy load routes** - Improve initial load time
5. **Use TypeScript for API responses** - Catch errors at compile time

---

## Security & Secrets

### Secret Management Rules

1. **Never commit secrets** - Use `.env` files (gitignored)
2. **Never log secrets** - Redact in logging configuration
3. **Never expose in client bundles** - Only `VITE_` prefixed vars are exposed
4. **Rotate regularly** - Especially after team changes
5. **Use different secrets per environment** - Dev/staging/production

### Environment Variables Template

```bash
# .env.example (commit this, not .env)
# Database
DATABASE_URL=postgresql://user:password@localhost:5432/dbname?sslmode=require

# Redis
REDIS_URL=redis://localhost:6379

# Authentication
JWT_SECRET=           # Generate: openssl rand -base64 32
SESSION_SECRET=       # Generate: openssl rand -base64 32
ARGON2_SECRET=        # Generate: openssl rand -base64 32

# External Services
RESEND_API_KEY=
LIVEKIT_API_KEY=
LIVEKIT_API_SECRET=

# Frontend (VITE_ prefix exposes to client)
VITE_API_URL=http://localhost:3000
VITE_WS_URL=ws://localhost:3000
```

### Encryption Requirements

| Data Type | At Rest | In Transit | Key Length |
|-----------|---------|------------|------------|
| Passwords | Argon2id | HTTPS | N/A (hash) |
| PII/PHI | AES-256-GCM | TLS 1.2+ | 256-bit |
| Session tokens | N/A | HTTPS | 256-bit |
| API keys | AES-256-GCM | HTTPS | 256-bit |

### Password Hashing

```typescript
import { hash, verify } from '@node-rs/argon2';

// Hash password
const hashedPassword = await hash(password, {
  memoryCost: 65536,    // 64MB
  timeCost: 3,          // 3 iterations
  parallelism: 4,       // 4 threads
  secret: Buffer.from(process.env.ARGON2_SECRET!)
});

// Verify password
const isValid = await verify(hashedPassword, password, {
  secret: Buffer.from(process.env.ARGON2_SECRET!)
});
```

### Secure Token Generation

```typescript
import { nanoid } from 'nanoid';
import { randomUUID } from 'crypto';

// Public-facing IDs (URLs, API responses)
const publicId = nanoid(21);  // ~128 bits of entropy

// Internal IDs with timestamp (audit trails)
const auditId = randomUUID();  // UUID v4

// Session tokens (high entropy)
const sessionToken = nanoid(32);  // ~192 bits of entropy
```

---

## Compliance Quick Reference

### Framework Comparison

| Control | SOC 2 | HIPAA | PCI DSS | GDPR |
|---------|-------|-------|---------|------|
| MFA | Recommended | Required for PHI | Required | Risk-based |
| Encryption at rest | Required | Required | Required | Appropriate |
| Encryption in transit | TLS 1.2+ | TLS 1.2+ | TLS 1.2+ | TLS 1.2+ |
| Audit log retention | 1 year | 6 years | 1 year | As needed |
| Session timeout | Configurable | ≤15 min for PHI | ≤15 min | Risk-based |
| Password requirements | Complex | Complex | 12+ chars | Risk-based |
| Vulnerability scans | Quarterly | Regular | Quarterly | Regular |
| Pen testing | Annual | Recommended | Annual | Risk-based |

### Unified Approach: Apply Strictest Requirement

When building for multiple compliance frameworks, implement the strictest requirement:

- **Password**: 12+ characters (PCI DSS)
- **Session timeout**: 15 minutes (HIPAA/PCI DSS)
- **Log retention**: 6 years (HIPAA)
- **MFA**: Required (PCI DSS)
- **Encryption**: AES-256, TLS 1.3 preferred
- **Vulnerability scans**: Quarterly + after changes

### Audit Logging Requirements

```typescript
// Required fields for compliance
interface AuditLog {
  timestamp: string;      // ISO 8601
  eventType: string;      // login, logout, access, modify, delete
  userId: string;         // Who performed the action
  resourceType: string;   // What type of resource
  resourceId: string;     // Which specific resource
  action: string;         // What was done
  outcome: 'success' | 'failure';
  ipAddress: string;      // Where from
  userAgent: string;      // How (browser, API client)
  // NEVER include PII/PHI in logs - reference by ID only
}
```

### Data Subject Rights (GDPR)

Implement these endpoints for GDPR compliance:

```typescript
// Right to access
GET /api/user/data-export

// Right to rectification
PATCH /api/user/profile

// Right to erasure (with exceptions for legal requirements)
DELETE /api/user/account

// Right to portability
GET /api/user/data-export?format=json
```

---

## Production Checklist

### Pre-Deployment

#### Security
- [ ] All secrets in environment variables, not code
- [ ] TLS 1.2+ configured for all connections
- [ ] Security headers configured (Helmet)
- [ ] Rate limiting enabled
- [ ] CORS properly configured
- [ ] Input validation on all endpoints (Zod)
- [ ] SQL injection prevented (parameterized queries/Prisma)
- [ ] XSS prevention (output encoding)
- [ ] CSRF protection enabled
- [ ] Authentication properly implemented
- [ ] Authorization checks on all protected routes

#### Docker
- [ ] No `npx` in Dockerfiles
- [ ] No `|| true` hiding failures
- [ ] Multi-stage builds implemented
- [ ] Running as non-root user
- [ ] NODE_ENV=production set
- [ ] Exact version pins for critical deps
- [ ] Alpine dependencies installed (openssl)
- [ ] Health checks configured
- [ ] Resource limits set

#### Database
- [ ] Migrations tested in staging
- [ ] Backups configured and tested
- [ ] Connection pooling configured
- [ ] SSL/TLS enabled for connections
- [ ] Indexes optimized
- [ ] Prisma client regenerated

#### Monitoring
- [ ] Structured logging (JSON)
- [ ] Log aggregation configured
- [ ] Error tracking (Sentry)
- [ ] Health check endpoints
- [ ] Metrics collection (optional)
- [ ] Alerting configured

#### Compliance (if applicable)
- [ ] Audit logging implemented
- [ ] PII/PHI redaction in logs
- [ ] Data encryption at rest
- [ ] Session timeouts configured
- [ ] MFA implemented (if required)
- [ ] Data retention policies
- [ ] Incident response plan

### Post-Deployment

- [ ] Verify health checks passing
- [ ] Verify logs flowing correctly
- [ ] Run smoke tests
- [ ] Verify database migrations applied
- [ ] Check error rates
- [ ] Verify SSL certificate valid
- [ ] Test critical user flows

---

## Quick Command Reference

### Prisma

```bash
# Generate client (after schema changes)
npx prisma generate

# Development migration (creates + applies)
npx prisma migrate dev --name migration_name

# Production migration (applies existing)
npx prisma migrate deploy

# Schema push (no migration files)
npx prisma db push

# Seed database
npx prisma db seed

# Open database GUI
npx prisma studio

# Validate schema
npx prisma validate

# Reset database (DEV ONLY)
npx prisma migrate reset
```

### Docker

```bash
# Build with no cache
docker compose build --no-cache

# Start services
docker compose up -d

# View logs
docker compose logs -f api

# Execute command in container
docker compose exec api sh

# Run one-off command
docker compose run --rm api npm run migrate

# Clean up everything
docker compose down -v --rmi all

# Check resource usage
docker stats
```

### Git Hooks (Husky)

```bash
# Install
npm install -D husky lint-staged
npx husky install

# Add pre-commit hook
npx husky add .husky/pre-commit "npx lint-staged"

# lint-staged.config.js
module.exports = {
  '*.{ts,tsx}': ['eslint --fix', 'prettier --write'],
  'prisma/schema.prisma': ['npx prisma generate', 'npx prisma validate'],
};
```

### SSL/TLS (Certbot)

```bash
# Initial certificate
certbot certonly --webroot -w /var/www/certbot -d example.com

# Renew certificates
certbot renew

# Test renewal
certbot renew --dry-run
```

### Quick Debugging

```bash
# Check if port is in use
lsof -i :3000

# Check Node.js memory
node --v8-options | grep -i memory

# Test database connection
psql $DATABASE_URL -c "SELECT 1"

# Test Redis connection
redis-cli -u $REDIS_URL ping

# Check container health
docker inspect --format='{{.State.Health.Status}}' container_name
```

---

## Common Mistakes Summary

| Mistake | Impact | Solution |
|---------|--------|----------|
| Using `npx` in Docker | Version mismatch | Use `./node_modules/.bin/` |
| Using `\|\| true` | Hidden failures | Remove or use explicit error handling |
| Missing `prisma generate` | Type errors | Add to build process and pre-commit |
| `db push` in production | No migration history | Use `migrate deploy` |
| Secrets in code | Security breach | Use environment variables |
| No rate limiting | DoS vulnerability | Implement rate limiting |
| Logging PII/PHI | Compliance violation | Redact sensitive fields |
| devDependencies in prod | Missing tools | Move runtime CLIs to dependencies |
| Missing Alpine deps | Build failures | Install openssl, python3, make, g++ |
| No health checks | Silent failures | Add `/health` endpoint |
| No session timeout | Compliance violation | Implement ≤15 min timeout |
| Missing TLS | Data exposure | Enforce TLS 1.2+ |

---

## Additional Resources

- [Prisma Documentation](https://www.prisma.io/docs)
- [Fastify Documentation](https://fastify.dev/docs)
- [Docker Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Node.js Security Checklist](https://nodejs.org/en/docs/guides/security/)

---

*Last updated: December 2024*

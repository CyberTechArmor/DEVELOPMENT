# Docker Dependency Management & Build Auditing

## The Problem: `npx` Downloads Latest Versions

When using `npx <package>` in Docker (or anywhere), if the package isn't installed locally, **npx downloads the latest version** — not the version in your `package.json`.

### Real Example

```dockerfile
# In final stage of multi-stage build
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma
RUN npx prisma generate  # ⚠️ Downloads Prisma 7.x, not 5.x!
```

**What happened:**
- Project uses Prisma `^5.22.0`
- Prisma client was already generated in builder stage with 5.x
- Final stage runs `npx prisma generate` without local `prisma` CLI
- `npx` downloads Prisma 7.x (latest)
- Prisma 7.x has breaking schema changes → build fails

---

## Correct Terminology

| Informal Phrase | Correct Term |
|-----------------|--------------|
| "The versions don't match" | **Dependency version mismatch** |
| "npx grabbed the wrong one" | **npx resolved to latest instead of pinned version** |
| "Cleaning up the build" | **Build artifact deduplication** or **eliminating redundant build steps** |
| "Making sure it works" | **Build auditing** or **dependency auditing** |

---

## Best Practices

### 1. Never use `npx` for version-sensitive operations in Docker

```dockerfile
# ❌ Bad - npx may download different version
RUN npx prisma generate

# ✅ Good - use the installed version explicitly
RUN ./node_modules/.bin/prisma generate

# ✅ Also good - npm scripts use local versions
RUN npm run prisma:generate
```

### 2. Avoid redundant generation in multi-stage builds

If you generate artifacts in a builder stage and copy them, don't regenerate:

```dockerfile
# Builder stage
FROM node:20-alpine AS builder
RUN npm ci
RUN npx prisma generate  # Generated here with correct version

# Production stage
FROM node:20-alpine AS production
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma
# ❌ Don't run prisma generate again - it's already done!
```

### 3. Pin exact versions for critical dependencies

```json
{
  "dependencies": {
    "@prisma/client": "5.22.0"
  },
  "devDependencies": {
    "prisma": "5.22.0"
  }
}
```

Note: `^5.22.0` allows minor updates. For Docker reproducibility, consider exact versions.

---

## Prompting AI to Avoid This Issue

When working with AI on Docker builds, include these prompts:

### Before starting:
```
"Show me the package.json versions for [prisma/webpack/etc] before modifying the Dockerfile"
```

### When reviewing Dockerfiles:
```
"Check if any npx commands in the Dockerfile might resolve to different versions than package.json specifies"
```

### General audit prompt:
```
"Audit this Dockerfile for:
1. Redundant build steps (generating artifacts that are already copied)
2. npx commands that might download different versions
3. Version mismatches between stages"
```

---

## Running a Build Audit

### When to Audit

| Trigger | Why |
|---------|-----|
| Before major deploys | Catch issues before production |
| After dependency updates | New versions may have breaking changes |
| When builds suddenly fail | Version drift often the cause |
| Periodically (weekly/monthly) | Prevent drift from accumulating |

### How to Audit

#### 1. Check for version consistency

```bash
# See what versions are installed
npm ls prisma @prisma/client

# Compare to what npx would download
npm view prisma version  # Shows latest

# Check all outdated packages
npm outdated
```

#### 2. Audit Dockerfile for redundant steps

```bash
# Find all RUN commands
grep -n "^RUN" Dockerfile

# Look for patterns like:
# - generate/build commands after COPY --from
# - npx without corresponding npm install
# - duplicate npm ci/install commands
```

#### 3. Security audit

```bash
# Check for known vulnerabilities
npm audit

# Fix automatically where possible
npm audit fix

# For production, consider:
npm audit --production  # Only production deps
```

#### 4. Build verification checklist

```bash
# 1. Clean build from scratch
docker build --no-cache -t myapp:audit .

# 2. Verify the image works
docker run --rm myapp:audit npm run healthcheck

# 3. Check image for correct versions
docker run --rm myapp:audit npm ls prisma

# 4. Scan for vulnerabilities
docker scout cve myapp:audit  # If using Docker Scout
# or
trivy image myapp:audit       # Using Trivy
```

---

## Quick Audit Script

Save as `scripts/audit-build.sh`:

```bash
#!/bin/bash
set -e

echo "=== Dependency Version Check ==="
npm ls --depth=0

echo -e "\n=== Outdated Packages ==="
npm outdated || true

echo -e "\n=== Security Audit ==="
npm audit --production || true

echo -e "\n=== Dockerfile Analysis ==="
if [ -f Dockerfile ]; then
  echo "Checking for npx usage..."
  grep -n "npx" Dockerfile || echo "No npx commands found"

  echo -e "\nChecking for redundant operations..."
  # Count COPY --from and subsequent RUN commands
  grep -n "COPY --from\|RUN.*generate\|RUN.*build" Dockerfile
fi

echo -e "\n=== TypeScript Check ==="
npx tsc --noEmit || echo "Type errors found!"

echo -e "\n=== Audit Complete ==="
```

---

## Common Mistakes Summary

| Mistake | Solution |
|---------|----------|
| Using `npx` in Docker final stage | Use `./node_modules/.bin/` or npm scripts |
| Regenerating already-copied artifacts | Remove redundant RUN commands |
| Using `^` versions in Docker builds | Consider exact versions for reproducibility |
| Not auditing before deploy | Add audit script to CI/CD pipeline |
| Ignoring `npm audit` warnings | Review and fix or document accepted risks |

---

## Pre-Deploy Checklist

- [ ] `npm audit` shows no critical vulnerabilities (or documented exceptions)
- [ ] `npm outdated` reviewed — no unexpected major version drift
- [ ] Dockerfile has no `npx` commands without local installation
- [ ] No redundant build/generate steps after `COPY --from`
- [ ] `docker build --no-cache` succeeds
- [ ] Container starts and passes health check
- [ ] Versions inside container match `package.json`

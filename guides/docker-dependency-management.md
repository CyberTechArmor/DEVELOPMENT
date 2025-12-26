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
| "The tool isn't there in prod" | **Runtime dependency missing** (devDependencies excluded) |
| "Need to rebuild from scratch" | **Cache invalidation** or **clean rebuild** |

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

### 4. Use correct paths in shell scripts that run container commands

Shell scripts (install.sh, update.sh) that run `docker compose exec/run` face **two compounding issues**:

1. `npx` downloads latest version (not your pinned version)
2. Wrong working directory means Prisma can't find the schema

**Real Example:**

```bash
# ❌ Bad - multiple problems
docker compose run --rm api npx prisma migrate deploy

# Problems:
# 1. npx downloads Prisma 7.x (not the 5.x in package.json)
# 2. Runs from /app but schema is at /app/apps/api/prisma/schema.prisma
# 3. Even if version was right, "prisma" directory not found error
```

```bash
# ✅ Good - explicit paths solve both issues
docker compose run --rm api sh -c "cd /app/apps/api && /app/node_modules/.bin/prisma migrate deploy"

# Why this works:
# 1. Uses installed Prisma 5.x binary from node_modules
# 2. cd to correct directory where schema.prisma lives
# 3. Prisma finds schema at ./prisma/schema.prisma (relative to cwd)
```

**Key insight:** In monorepos, the working directory for Prisma commands must be where `prisma/schema.prisma` lives, not the root.

### 5. Put runtime-required CLIs in `dependencies`, not `devDependencies`

Production containers typically run `npm ci --omit=dev`, which **excludes devDependencies**. If a CLI tool is needed at runtime, it must be in `dependencies`.

**Real Example:**

```json
{
  "dependencies": {
    "@prisma/client": "^5.22.0"
  },
  "devDependencies": {
    "prisma": "^5.22.0"  // ⚠️ Won't be in production container!
  }
}
```

**What happens:**
- `npm ci --omit=dev` installs only `dependencies`
- `prisma` CLI is not installed in production
- `npx prisma migrate deploy` fails (binary not found)
- Or worse: `npx` downloads Prisma 7.x as a fallback

**The fix:**

```json
{
  "dependencies": {
    "@prisma/client": "^5.22.0",
    "prisma": "^5.22.0"  // ✅ Available in production for migrations
  }
}
```

**Rule of thumb:** If you run a command in production (migrations, seeding, etc.), its package belongs in `dependencies`.

| Tool | Typical Use | Where It Belongs |
|------|-------------|------------------|
| `prisma` | Migrations in prod | `dependencies` |
| `typescript` | Build only | `devDependencies` |
| `eslint` | Linting only | `devDependencies` |
| `tsx` | Dev server only | `devDependencies` |

**After fixing package.json**, you must rebuild from scratch:

```bash
# Docker layer cache has old npm ci result
docker build --no-cache -t myapp .
```

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

### When reviewing shell scripts:
```
"Check install.sh and update.sh for:
1. npx commands inside docker compose exec/run that might download wrong versions
2. Commands that assume wrong working directory (especially in monorepos)
3. Prisma commands - verify they run from the directory containing prisma/schema.prisma"
```

### When checking package.json:
```
"Check if any CLI tools in devDependencies are used in production:
1. Is 'prisma' in devDependencies but migrations run in production?
2. Are there any scripts in the Dockerfile or install.sh that use devDependency packages?
3. Does the Dockerfile use --omit=dev or --production flag?"
```

### General audit prompt:
```
"Audit this Dockerfile and shell scripts for:
1. Redundant build steps (generating artifacts that are already copied)
2. npx commands that might download different versions
3. Version mismatches between stages
4. Working directory issues in docker compose exec/run commands
5. CLI tools in devDependencies that are needed at runtime"
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
| `npx` in shell scripts with `docker compose` | Use full path: `/app/node_modules/.bin/prisma` |
| Wrong working directory in container | Use `sh -c "cd /correct/path && command"` |
| Prisma can't find schema in monorepo | `cd` to directory containing `prisma/schema.prisma` first |
| Runtime CLI in `devDependencies` | Move to `dependencies` if used in production |
| Cache not cleared after package.json fix | Rebuild with `docker build --no-cache` |

---

## Pre-Deploy Checklist

- [ ] `npm audit` shows no critical vulnerabilities (or documented exceptions)
- [ ] `npm outdated` reviewed — no unexpected major version drift
- [ ] Dockerfile has no `npx` commands without local installation
- [ ] No redundant build/generate steps after `COPY --from`
- [ ] Shell scripts use `/app/node_modules/.bin/` instead of `npx`
- [ ] Shell scripts `cd` to correct directory before running Prisma commands
- [ ] Runtime CLIs (prisma, etc.) are in `dependencies`, not `devDependencies`
- [ ] `docker build --no-cache` succeeds (especially after package.json changes)
- [ ] Container starts and passes health check
- [ ] Versions inside container match `package.json`

# Docker Dependency Management & Build Auditing

> **Scope:** This guide focuses on **production** container builds. Development environments have different considerations (hot reloading, debugging tools, etc.).

---

## Production vs Development: Key Differences

| Aspect | Development | Production |
|--------|-------------|------------|
| `npm install` | Full install (all deps) | `npm ci --omit=dev` |
| Source maps | Enabled | Disabled (security) |
| Debug tools | Included | Excluded |
| Environment | `NODE_ENV=development` | `NODE_ENV=production` |
| Image size | Not critical | Minimize |
| Build cache | Aggressive | Clean builds for releases |

**Critical production flags:**
```dockerfile
# Production Dockerfile patterns
ENV NODE_ENV=production
RUN npm ci --omit=dev    # Excludes devDependencies
```

---

## Version Pinning Strategy & Risk Assessment

### Pinning Strategies Compared

| Strategy | Example | Reproducibility | Security Updates | Risk Level |
|----------|---------|-----------------|------------------|------------|
| **Exact** | `"5.22.0"` | ✅ Highest | ❌ Manual | Low (stable) |
| **Patch** | `"~5.22.0"` | Medium | ⚠️ Patch only | Low |
| **Minor** | `"^5.22.0"` | Lower | ⚠️ Minor+Patch | Medium |
| **Latest** | `"*"` or `npx` | ❌ None | ✅ Automatic | **High** |

### When to Use Each Strategy

**Exact versions (`5.22.0`)** — Use for:
- Production Docker builds (reproducibility critical)
- CI/CD pipelines
- When a newer version has known breaking changes

**Caret versions (`^5.22.0`)** — Acceptable for:
- Development environments
- Non-critical dependencies
- When you have good test coverage

**Never use in production:**
- `*` or `latest` tags
- `npx` without local installation (downloads latest)

### Documenting Version Decisions

When pinning to a non-latest version, document the reason:

```json
{
  "dependencies": {
    "@prisma/client": "5.22.0",
    "prisma": "5.22.0"
  },
  "_versionNotes": {
    "prisma": "Pinned to 5.x - Prisma 6.x/7.x have breaking schema changes (removed 'url' from datasource). Upgrade requires schema migration. See: https://www.prisma.io/docs/orm/more/upgrade-guides"
  }
}
```

Or in a `VERSIONS.md` file:

```markdown
## Pinned Versions

| Package | Version | Reason | Risk if Upgraded | Review Date |
|---------|---------|--------|------------------|-------------|
| prisma | 5.22.0 | 6.x/7.x break schema format | Build fails | 2025-03-01 |
| node | 20-alpine | LTS until 2026-04 | None expected | 2025-06-01 |
```

### Security Implications of Pinned Versions

**Risks of staying on older versions:**
- Missing security patches
- Accumulating technical debt
- Harder upgrades over time

**Mitigation:**
1. Run `npm audit` regularly (weekly minimum)
2. Subscribe to security advisories for critical deps
3. Schedule quarterly version reviews
4. Document a clear upgrade path

```bash
# Check for security issues in production deps only
npm audit --production

# See what's outdated
npm outdated

# Check specific package changelog
npm view prisma versions --json | tail -20
```

---

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
| "It's in the wrong node_modules" | **Dependency not hoisted** (npm workspaces) |
| "The lockfile is stale" | **Lockfile out of sync** with package.json |
| "It's still in dev mode" | **Production environment misconfiguration** |
| "Why is it using an old version?" | **Intentional version pinning** (document the reason!) |
| "Build takes forever" | **npx downloading on every build** (use local binary) |
| "Prisma can't find libssl" | **Missing Alpine system dependency** (add openssl) |
| "Password authentication failed" | **Volume credential mismatch** (old creds in volume) |
| "Works first time but fails on re-install" | **Stale Docker volumes** (need `down -v`) |

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

### 6. Force hoisting in npm workspaces (monorepos)

In npm workspaces, dependencies are installed where they're declared. If `prisma` is only in `apps/api/package.json`, it installs to `apps/api/node_modules/.bin/prisma` — **not** the root.

**Real Example:**

```
monorepo/
├── package.json              # No prisma here
├── node_modules/             # No prisma binary here!
└── apps/
    └── api/
        ├── package.json      # "prisma": "^5.22.0"
        └── node_modules/
            └── .bin/
                └── prisma    # Prisma is here, not accessible from /app
```

**The Docker problem:**

```dockerfile
# Dockerfile copies from root
COPY --from=builder /app/node_modules ./node_modules

# But prisma is in /app/apps/api/node_modules/.bin/prisma
# This path doesn't exist in the production container!
RUN /app/node_modules/.bin/prisma migrate deploy  # ❌ Not found
```

**The fix — add to root package.json:**

```json
{
  "name": "monorepo",
  "workspaces": ["apps/*"],
  "dependencies": {
    "prisma": "^5.22.0"
  }
}
```

This forces npm to **hoist** prisma to the root `node_modules/`, making it available at `/app/node_modules/.bin/prisma`.

**Verification:**

```bash
# After npm install, check where prisma lives
ls node_modules/.bin/prisma  # Should exist at root

# Or check package-lock.json
grep -A2 '"node_modules/prisma"' package-lock.json
```

**Remember:** After modifying root `package.json`, regenerate the lockfile:

```bash
npm install --package-lock-only --ignore-scripts
```

### 7. Install required system dependencies in Alpine images

Alpine Linux is minimal — many packages that Node.js tools expect are missing. Prisma, in particular, requires OpenSSL.

**The error:**
```
Prisma failed to detect the libssl/openssl required for the native binary target
```

**The fix — install openssl in all stages that use Prisma:**

```dockerfile
# Builder stage - needs openssl for prisma generate
FROM node:20-alpine AS builder
RUN apk add --no-cache python3 make g++ curl openssl
# ... npm ci, prisma generate, etc.

# Production stage - needs openssl for prisma runtime
FROM node:20-alpine AS production
RUN apk add --no-cache curl openssl
# ... copy from builder, etc.
```

**Common Alpine dependencies by tool:**

| Tool | Required Packages | Why |
|------|-------------------|-----|
| Prisma | `openssl` | Native binary requires libssl |
| bcrypt/argon2 | `python3 make g++` | Native compilation |
| node-gyp | `python3 make g++` | Native addon building |
| Health checks | `curl` or `wget` | HTTP health probes |
| Sharp (images) | `vips-dev` | Image processing |

**Build time savings:**

Using `npx prisma generate` in Docker downloads Prisma each time (~4-5 minutes). Using the local binary is instant:

```dockerfile
# ❌ Bad - downloads prisma every build (~4-5 min)
RUN npx prisma generate

# ✅ Good - uses already-installed prisma (instant)
RUN ./node_modules/.bin/prisma generate

# ✅ In monorepo - adjust path accordingly
RUN cd apps/api && ../../node_modules/.bin/prisma generate
```

### 8. Handle Docker volume credential persistence

Docker volumes persist data across container restarts and rebuilds. For databases like PostgreSQL, **credentials are stored when the volume is first initialized** — subsequent rebuilds with new credentials will fail.

**The problem:**

```bash
# First install - generates random password, stores in volume
DB_PASSWORD=abc123  # Saved to postgres_data volume

# Re-install - generates NEW random password
DB_PASSWORD=xyz789  # Volume still has abc123 → authentication fails!
```

**The error:**
```
FATAL: password authentication failed for user "postgres"
```

**The fix — clean volumes before fresh install:**

```bash
# In install.sh, before building images:
docker compose down -v 2>/dev/null || true
```

The `-v` flag removes named volumes, ensuring credentials are re-initialized with the new password.

**When to clean volumes vs preserve them:**

| Scenario | Action | Why |
|----------|--------|-----|
| Fresh install | `docker compose down -v` | Start clean |
| Re-install with new creds | `docker compose down -v` | Old creds in volume |
| Update with same creds | `docker compose down` (no -v) | Preserve data |
| Production backup | Never use `-v` without backup | Data loss! |

**Install script pattern:**

```bash
# install.sh - for fresh installations
log_info "Cleaning up any existing containers and volumes..."
docker compose down -v 2>/dev/null || true

# Generate new random credentials
DB_PASSWORD=$(openssl rand -base64 32)
echo "DB_PASSWORD=$DB_PASSWORD" >> .env

# Now build and start
docker compose up -d
```

**Update script pattern:**

```bash
# update.sh - preserves data
log_info "Stopping containers (keeping volumes)..."
docker compose down  # No -v flag!

# Pull/build new images
docker compose pull
docker compose up -d
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

### For npm workspaces (monorepos):
```
"Check for dependency hoisting issues:
1. Is this a monorepo with npm workspaces?
2. Are CLI tools like prisma only in a workspace package.json, not the root?
3. Does the Dockerfile copy from root node_modules but expect workspace binaries?
4. Verify package-lock.json shows the tool under 'node_modules/[tool]' at root level"
```

### General audit prompt:
```
"Audit this Dockerfile and shell scripts for:
1. Redundant build steps (generating artifacts that are already copied)
2. npx commands that might download different versions
3. Version mismatches between stages
4. Working directory issues in docker compose exec/run commands
5. CLI tools in devDependencies that are needed at runtime
6. Workspace dependencies not hoisted to root (monorepos)
7. package-lock.json out of sync with package.json changes
8. NODE_ENV=production is set in the final stage
9. npm ci --omit=dev is used (not npm install)
10. No debug tools or source maps in production image"
```

### Production readiness prompt:
```
"Verify this is production-ready:
1. Is NODE_ENV set to 'production'?
2. Are devDependencies excluded (--omit=dev)?
3. Are there any localhost URLs in environment variables?
4. Are pinned versions documented with upgrade paths?
5. Has npm audit --production been run recently?
6. Are source maps disabled or excluded?"
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
| CLI in workspace but not root (monorepo) | Add to root `package.json` to force hoisting |
| Lockfile not regenerated after fix | Run `npm install --package-lock-only --ignore-scripts` |
| `NODE_ENV` not set to production | Add `ENV NODE_ENV=production` to Dockerfile |
| Using `npm install` instead of `npm ci` | Use `npm ci --omit=dev` for reproducible prod builds |
| Pinned version without documentation | Add `_versionNotes` in package.json or VERSIONS.md |
| Source maps in production | Set `sourceMap: false` or exclude from COPY |
| Missing OpenSSL in Alpine for Prisma | Add `apk add --no-cache openssl` in Dockerfile |
| Using `npx` slows build by 4-5 min | Use `./node_modules/.bin/prisma` instead |
| DB auth fails after re-install | Old volume has old creds; use `docker compose down -v` |
| `down -v` in update script | Never use `-v` in updates — destroys data! |

---

## Pre-Deploy Checklist

### Security & Dependencies
- [ ] `npm audit --production` shows no critical vulnerabilities (or documented exceptions)
- [ ] `npm outdated` reviewed — no unexpected major version drift
- [ ] Pinned versions documented with rationale (see Version Pinning section)
- [ ] Security advisory subscriptions active for critical deps

### Dockerfile & Build
- [ ] `NODE_ENV=production` is set
- [ ] Uses `npm ci --omit=dev` (not `npm install`)
- [ ] No `npx` commands — use `./node_modules/.bin/` paths instead
- [ ] No redundant build/generate steps after `COPY --from`
- [ ] Alpine images have required system deps (`openssl` for Prisma, etc.)
- [ ] Source maps disabled or excluded from final image
- [ ] No debug tools or dev utilities in production image

### Shell Scripts & Runtime
- [ ] Shell scripts use `/app/node_modules/.bin/` instead of `npx`
- [ ] Shell scripts `cd` to correct directory before running Prisma commands
- [ ] Runtime CLIs (prisma, etc.) are in `dependencies`, not `devDependencies`
- [ ] For monorepos: CLI tools are in root `package.json` (not just workspace)

### Build Verification
- [ ] `package-lock.json` regenerated after any `package.json` changes
- [ ] `docker build --no-cache` succeeds (especially after package.json changes)
- [ ] Container starts and passes health check
- [ ] `NODE_ENV` inside container is `production`
- [ ] Versions inside container match `package.json`
- [ ] CLI binaries exist at expected paths inside container
- [ ] No devDependencies present in production container

### Volume & Credential Management
- [ ] Install script uses `docker compose down -v` for clean state
- [ ] Update script does NOT use `-v` (preserves data)
- [ ] Database credentials match between `.env` and existing volumes
- [ ] Production volumes backed up before any destructive operations

### Verification Commands

```bash
# Verify production environment
docker run --rm myapp printenv NODE_ENV  # Should print: production

# Check no devDependencies leaked
docker run --rm myapp npm ls --omit=dev 2>&1 | grep -E "typescript|eslint|vitest"
# Should return nothing

# Verify correct versions
docker run --rm myapp npm ls prisma @prisma/client

# Check binary paths
docker run --rm myapp ls -la /app/node_modules/.bin/prisma
```

---

## Production Environment Validation

Always validate these settings before production deploy:

```bash
#!/bin/bash
# scripts/validate-production.sh

echo "=== Production Environment Validation ==="

# Check NODE_ENV
if [ "$NODE_ENV" != "production" ]; then
  echo "❌ NODE_ENV is not 'production': $NODE_ENV"
  exit 1
fi
echo "✅ NODE_ENV=production"

# Check for debug flags
if [ -n "$DEBUG" ]; then
  echo "⚠️  DEBUG is set: $DEBUG"
fi

# Check Prisma database URL is not localhost
if echo "$DATABASE_URL" | grep -q "localhost"; then
  echo "❌ DATABASE_URL contains localhost - not production!"
  exit 1
fi
echo "✅ DATABASE_URL appears to be production"

# Verify no dev dependencies
if npm ls 2>/dev/null | grep -qE "devDependencies"; then
  echo "⚠️  devDependencies may be present"
fi

echo "=== Validation Complete ==="
```

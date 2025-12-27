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
| "tsx/ts-node not found" | **TypeScript runner missing** (compile to JS or install tsx globally) |
| "No migrations found" | **Wrong Prisma command** (use `db push` if no migrations dir) |
| "tsc can't find module" | **Standalone compilation failure** (tsc can't resolve imports without tsconfig) |
| "tsx can't find @prisma/client" | **Monorepo module resolution** (create symlinks, NODE_PATH won't work) |
| "@prisma/client did not initialize yet" | **Generated client wrong location** (copy .prisma to root node_modules) |
| "Build step passed but it failed" | **Silent failure** (command has `\|\| true` masking errors) |
| "Build killed" or OOM | **Out of memory** (add swap space, limit Node memory) |
| "Build takes forever on VPS" | **Resource constrained** (1-core + low RAM needs optimization) |
| "No renewals were attempted" (certbot) | **Entrypoint override needed** (use `--entrypoint ""` for certonly) |
| "host not found in upstream" (nginx) | **Container not running** (start dependent containers first) |
| "ENCRYPTION_KEY must be 64 characters" | **Secret too short** (generate 64+ char key, not 32) |

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

### 9. Handle TypeScript scripts in production

TypeScript runners like `tsx`, `ts-node`, and `ts-jest` are typically devDependencies. If your project has TypeScript scripts (like `prisma/seed.ts`) that need to run in production, you have several options.

**The error:**
```
sh: tsx: not found
```

**The problem:**
```json
{
  "prisma": {
    "seed": "tsx prisma/seed.ts"  // tsx is devDependency!
  }
}
```

#### Option A: Install tsx globally in production (Recommended for scripts with imports)

If your TypeScript script has imports from your project, **standalone tsc compilation won't work**:

```typescript
// prisma/seed.ts
import { hashPassword } from '../src/utils/auth';  // ← tsc can't resolve this!
import { UserRole } from '../src/types';            // ← or this!
```

**Why tsc fails:**
```bash
# This FAILS - tsc can't resolve imports without full project context
tsc prisma/seed.ts --outDir prisma
# Error: Cannot find module '../src/utils/auth'
```

**The fix — install tsx globally in production image:**

```dockerfile
# Production stage
FROM node:20-alpine AS production

# Install tsx globally for running TypeScript scripts with imports
RUN npm install -g tsx

# Copy the source files that seed.ts imports
COPY --from=builder /app/apps/api/src/types ./apps/api/src/types
COPY --from=builder /app/apps/api/src/utils ./apps/api/src/utils

# Now tsx can run seed.ts with all its imports
```

**Run in production:**
```bash
# In install.sh or docker compose
docker compose run --rm api sh -c "cd /app && tsx apps/api/prisma/seed.ts"
```

#### Monorepo module resolution: Create symlinks (not NODE_PATH)

In monorepos, tsx fails to find npm packages like `@prisma/client` because modules are installed at the root `node_modules/`, not in the subdirectory where the script lives.

**The error:**
```
Error: Cannot find module '@prisma/client'
Require stack:
- /app/apps/api/prisma/seed.ts
```

**Why it happens:**
```
monorepo/
├── node_modules/           ← @prisma/client is here
│   └── @prisma/client/
└── apps/
    └── api/
        ├── node_modules/   ← tsx looks here first!
        │   └── .prisma/    ← (only generated client here)
        └── prisma/
            └── seed.ts     ← tsx resolves relative to THIS file
```

tsx resolves modules relative to the **file location**, not the current working directory. Even with `cd /app`, tsx still looks for modules starting from `/app/apps/api/`.

**❌ NODE_PATH does NOT work with tsx:**
```bash
# This FAILS - tsx ignores NODE_PATH
docker compose run --rm api sh -c "cd /app && NODE_PATH=/app/node_modules tsx apps/api/prisma/seed.ts"
# Still gets: Cannot find module '@prisma/client'
```

tsx has its own module resolution that bypasses Node's `NODE_PATH` environment variable.

**✅ The fix — create symlinks in Dockerfile:**

```dockerfile
# Copy generated Prisma client (created by prisma generate)
COPY --from=api-builder /app/apps/api/node_modules/.prisma ./apps/api/node_modules/.prisma

# Create symlinks for seed script module resolution (tsx resolves relative to file location)
RUN mkdir -p /app/apps/api/node_modules/@prisma && \
    ln -s /app/node_modules/@prisma/client /app/apps/api/node_modules/@prisma/client && \
    ln -s /app/node_modules/@node-rs /app/apps/api/node_modules/@node-rs
```

**Why symlinks work:**
1. tsx looks for `@prisma/client` at `/app/apps/api/node_modules/@prisma/client`
2. Symlink points to `/app/node_modules/@prisma/client`
3. tsx follows the symlink and finds the module

#### Critical: Copy `.prisma/client` to root node_modules

Even after fixing `@prisma/client` resolution, you may get:

```
Error: @prisma/client did not initialize yet. Please run "prisma generate"
```

**Why this happens:**
- `@prisma/client` is found at `/app/apps/api/node_modules/@prisma/client`
- But internally, `@prisma/client` imports from `.prisma/client` relative to **root** node_modules
- It looks at `/app/node_modules/.prisma/client/` (doesn't exist!)
- The generated client is at `/app/apps/api/node_modules/.prisma/client/`

**The fix — copy generated client to root:**

```dockerfile
# Copy generated .prisma/client to ROOT node_modules (where @prisma/client expects it)
COPY --from=api-builder /app/apps/api/node_modules/.prisma ./node_modules/.prisma

# Also copy to apps/api for completeness
COPY --from=api-builder /app/apps/api/node_modules/.prisma ./apps/api/node_modules/.prisma
```

**Complete monorepo Prisma setup:**
```dockerfile
# 1. Copy root node_modules (has @prisma/client if hoisted)
COPY --from=deps /app/node_modules ./node_modules

# 2. Copy generated .prisma client to root (critical!)
COPY --from=api-builder /app/apps/api/node_modules/.prisma ./node_modules/.prisma

# 3. Copy workspace node_modules (has .prisma if not hoisted)
COPY --from=api-builder /app/apps/api/node_modules ./apps/api/node_modules

# 4. Ensure @prisma/client is in workspace for tsx resolution
RUN if [ ! -d /app/apps/api/node_modules/@prisma/client ]; then \
      mkdir -p /app/apps/api/node_modules/@prisma && \
      cp -r /app/node_modules/@prisma/client /app/apps/api/node_modules/@prisma/; \
    fi
```

**Run the seed script:**
```bash
docker compose run --rm api sh -c "cd /app && tsx apps/api/prisma/seed.ts"
```

#### Alternative: Regenerate Prisma in production image

If copying `.prisma/client` between stages causes initialization errors, regenerate it in the production image:

```dockerfile
# Copy root node_modules
COPY --from=deps /app/node_modules ./node_modules

# Copy api dist, prisma schema, and types
COPY --from=api-builder /app/apps/api/dist ./apps/api/dist
COPY --from=api-builder /app/apps/api/prisma ./apps/api/prisma
COPY --from=api-builder /app/apps/api/src/types ./apps/api/src/types

# Regenerate Prisma client in production (ensures correct paths)
RUN cd /app/apps/api && /app/node_modules/.bin/prisma generate

# Create symlinks for tsx module resolution
RUN mkdir -p /app/apps/api/node_modules/@prisma /app/apps/api/node_modules/.prisma && \
    ln -sf /app/node_modules/@prisma/client /app/apps/api/node_modules/@prisma/client && \
    ln -sf /app/node_modules/.prisma/client /app/apps/api/node_modules/.prisma/client && \
    ln -sf /app/node_modules/@node-rs /app/apps/api/node_modules/@node-rs
```

**Why this works:**
- `prisma generate` creates `.prisma/client` with correct paths for the current environment
- No path mismatch issues from copying between stages
- Symlinks let tsx find modules while using the properly initialized client

**Trade-off:** Adds ~10-30 seconds to build time but eliminates initialization errors.

**Important: Clear Docker cache after Dockerfile changes**

Docker aggressively caches layers. If you modify the Dockerfile but see the old behavior, force a rebuild:

```bash
# Clear all Docker build cache
docker system prune -af

# Or rebuild without cache
docker compose build --no-cache
```

For fresh installs, use `--no-cache` to ensure the latest Dockerfile is used.

#### Option B: Compile during build (Only for scripts WITHOUT imports)

If your TypeScript script is self-contained with no project imports:

```dockerfile
# After building the main application, compile standalone scripts
RUN cd apps/api && ../../node_modules/.bin/tsc prisma/seed.ts \
    --outDir prisma \
    --esModuleInterop \
    --skipLibCheck \
    --resolveJsonModule
```

**Then run with Node:**
```bash
node prisma/seed.js
```

#### Option C: Write production scripts in JavaScript

Keep TypeScript out of production entirely:
```javascript
// prisma/seed.js - works without tsx or compilation
const { PrismaClient } = require('@prisma/client');
const bcrypt = require('bcrypt');  // Use npm package, not project imports

const prisma = new PrismaClient();
// ...
```

**Comparison:**

| Approach | Works with imports | Build complexity | Runtime overhead |
|----------|-------------------|------------------|------------------|
| tsx global | ✅ Yes | Low | Minimal (~50MB) |
| tsc compile | ❌ No (standalone only) | Medium | None |
| JavaScript | ❌ N/A | None | None |

**Common TypeScript scripts that need compilation:**

| Script | Location | Has Imports? | Recommended Approach |
|--------|----------|--------------|---------------------|
| Seed script | `prisma/seed.ts` | Usually yes | tsx global |
| DB migrations | `scripts/migrate.ts` | Varies | Check imports first |
| Admin scripts | `scripts/*.ts` | Usually yes | tsx global |

### 10. Avoid silent failures with `|| true`

Using `|| true` in Dockerfile RUN commands **masks failures** — the build succeeds but the step didn't actually work.

**The problem:**
```dockerfile
# ❌ Bad - silently fails, build continues
RUN tsc prisma/seed.ts --outDir prisma || true

# What happens:
# 1. tsc fails (can't resolve imports)
# 2. || true makes exit code 0
# 3. Build continues "successfully"
# 4. seed.js doesn't exist at runtime → crash
```

**Why this is dangerous:**
- Build appears to succeed
- Error only discovered at runtime (when seed fails)
- Hard to debug — no error message in build logs
- Creates false confidence in the build

**The fix — fail fast or handle explicitly:**

```dockerfile
# ✅ Good - fails immediately if tsc fails
RUN tsc prisma/seed.ts --outDir prisma

# ✅ Also good - explicit fallback with logging
RUN tsc prisma/seed.ts --outDir prisma \
    || (echo "WARNING: seed.ts compilation failed, using tsx at runtime" && exit 0)

# ✅ Best - use the right tool (tsx for scripts with imports)
RUN npm install -g tsx
# Then run with tsx at runtime instead of compiling
```

**When `|| true` is acceptable:**

| Scenario | Example | Why OK |
|----------|---------|--------|
| Optional cleanup | `rm -rf temp || true` | Non-critical |
| Check existence | `docker compose down -v 2>/dev/null \|\| true` | May not exist |
| First-run detection | `test -f .initialized \|\| true` | Expected to fail |

**Never use `|| true` for:**
- Compilation steps
- Dependency installation
- Critical file operations
- Migrations or data operations

### 11. Optimize for low-resource VPS (≤2GB RAM, ≤2 cores)

Building Node.js applications with TypeScript compilation requires significant memory. On low-resource VPS systems, builds can OOM (out of memory) or freeze.

**Symptoms of resource constraints:**
```
FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed - JavaScript heap out of memory
Killed
Build process hangs indefinitely
```

#### Hardware detection

Add hardware detection to install scripts:

```bash
# Get total RAM in MB
get_total_ram_mb() {
    grep MemTotal /proc/meminfo | awk '{print int($2/1024)}'
}

# Get CPU cores
get_cpu_cores() {
    grep -c ^processor /proc/cpuinfo
}

# Check if low-resource system
is_low_resource_system() {
    local ram_mb=$(get_total_ram_mb)
    local cores=$(get_cpu_cores)
    [ "$ram_mb" -le 2048 ] || [ "$cores" -le 1 ]
}
```

#### Swap space configuration

For systems with ≤2GB RAM, configure swap space (2x RAM):

```bash
configure_swap() {
    local ram_mb=$(get_total_ram_mb)
    local swap_mb=$((ram_mb * 2))

    # Create swap file
    sudo fallocate -l ${swap_mb}M /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile

    # Persist across reboots
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

    # Optimize swappiness for builds
    sudo sysctl vm.swappiness=60
}
```

#### Limit Node.js memory during builds

```dockerfile
# Limit Node.js memory to prevent OOM
ENV NODE_OPTIONS="--max-old-space-size=512"

# For 1GB RAM VPS, this prevents Node from consuming all memory
# Build will be slower but won't OOM
```

#### Docker daemon optimization

Create `/etc/docker/daemon.json`:
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
```

#### Clear caches before build

```bash
# Clear page cache on very low memory systems
if [ "$(get_total_ram_mb)" -le 1024 ]; then
    sync && echo 3 | sudo tee /proc/sys/vm/drop_caches > /dev/null
fi
```

#### Expected build times on low-resource VPS

| Hardware | npm ci | Prisma generate | TypeScript build | Total |
|----------|--------|-----------------|------------------|-------|
| 1-core, 1GB | ~45s | ~10min | ~12min | ~25min |
| 2-core, 2GB | ~30s | ~4min | ~5min | ~10min |
| 4-core, 4GB | ~20s | ~1min | ~2min | ~4min |

**Optimization impact:**

| Optimization | Effect |
|--------------|--------|
| 2GB swap on 1GB RAM | Prevents OOM kills |
| `--max-old-space-size=512` | Limits Node memory, prevents freeze |
| Docker layer caching (no `--no-cache`) | Rebuilds drop from 25min to 2-3min |
| Pre-built images from registry | Eliminates build entirely on VPS |

#### Alternative: Build elsewhere, pull images

For very constrained systems, build on a more powerful machine:

```bash
# On powerful machine:
docker build -t registry.example.com/myapp:latest .
docker push registry.example.com/myapp:latest

# On VPS:
docker pull registry.example.com/myapp:latest
# Instant — no build required
```

This eliminates the build bottleneck entirely on low-resource VPS.

### 12. Install and uninstall script patterns

Production install scripts should support CLI options for flexibility while having sensible defaults.

#### Install script options

```bash
#!/bin/bash

# Command line options with defaults
NO_CACHE=false
SKIP_HARDWARE_CHECK=false

# Parse command line arguments
while [[ $# -gt 0 ]]; do
    case $1 in
        --no-cache)
            NO_CACHE=true
            shift
            ;;
        --skip-hardware-check)
            SKIP_HARDWARE_CHECK=true
            shift
            ;;
        --help|-h)
            echo "Usage: $0 [OPTIONS]"
            echo ""
            echo "Options:"
            echo "  --no-cache            Force Docker to rebuild without using cache"
            echo "  --skip-hardware-check Skip hardware detection and optimization"
            echo "  --help, -h            Show this help message"
            exit 0
            ;;
        *)
            echo "Unknown option: $1"
            exit 1
            ;;
    esac
done
```

**Key principles:**
- `--no-cache` should be **optional, not default** — leverage Docker cache for fast rebuilds
- Include hardware detection that users can skip if needed
- Always provide `--help` output

**Build with optional cache bypass:**
```bash
build_images() {
    local build_opts=""
    if [ "$NO_CACHE" = true ]; then
        build_opts="--no-cache"
        log_info "Building without cache..."
    fi

    docker compose build $build_opts
}
```

**Secret generation — check application requirements:**
```bash
# Generate secrets with configurable length
generate_secret() {
    local length=${1:-32}
    openssl rand -base64 $((length * 2)) | tr -dc 'a-zA-Z0-9' | head -c $length
}

# Common secret lengths (check your app's requirements!)
jwt_secret=$(generate_secret 32)       # JWT typically 32+
encryption_key=$(generate_secret 64)   # AES-256 needs 64 hex chars
api_key=$(generate_secret 48)          # API keys vary
```

**Common mistake:** Generating 32-character secrets when the application requires 64. Always check error messages like "ENCRYPTION_KEY must be at least 64 characters".

#### Uninstall script with graduated cleanup

Uninstall scripts should offer levels of cleanup, with the safest option as default:

```bash
#!/bin/bash

REMOVE_DATA=false
REMOVE_IMAGES=false
REMOVE_BUILD_CACHE=false
FORCE=false

while [[ $# -gt 0 ]]; do
    case $1 in
        --remove-data)
            REMOVE_DATA=true
            shift
            ;;
        --remove-images)
            REMOVE_IMAGES=true
            shift
            ;;
        --remove-build-cache)
            REMOVE_BUILD_CACHE=true
            shift
            ;;
        --all)
            REMOVE_DATA=true
            REMOVE_IMAGES=true
            REMOVE_BUILD_CACHE=true
            shift
            ;;
        -f|--force)
            FORCE=true
            shift
            ;;
        -h|--help)
            show_help
            exit 0
            ;;
    esac
done
```

**Cleanup levels:**

| Option | What it removes | Safe for re-install? |
|--------|-----------------|---------------------|
| (default) | Containers, networks, install dir | ✅ Yes |
| `--remove-data` | + Database volumes | ⚠️ Data lost |
| `--remove-images` | + Docker images | ✅ Yes (slow rebuild) |
| `--remove-build-cache` | + Build cache | ✅ Yes (slow rebuild) |
| `--all` | Everything | ⚠️ Data lost, slow rebuild |

**Remove build cache function:**
```bash
remove_build_cache() {
    if [[ "$REMOVE_BUILD_CACHE" != true ]]; then
        return 0
    fi

    log_info "Removing Docker build cache..."
    docker builder prune -af 2>/dev/null || true
    docker system prune -f 2>/dev/null || true
    log_success "Docker build cache removed"
}
```

**Completion message with next steps:**
```bash
print_completion() {
    echo ""
    echo "Uninstall completed:"
    if [[ "$REMOVE_DATA" == true ]]; then
        echo "    ✓ Database volumes removed"
    fi
    if [[ "$REMOVE_IMAGES" == true ]]; then
        echo "    ✓ Docker images removed"
    fi
    if [[ "$REMOVE_BUILD_CACHE" == true ]]; then
        echo "    ✓ Docker build cache removed"
    fi
    echo "    ✓ Installation directory removed"
    echo ""

    # Show what's still there
    if [[ "$REMOVE_DATA" != true ]]; then
        echo "Note: Database volumes preserved. To remove:"
        echo "  docker volume ls | grep appname"
        echo "  docker volume rm <volume_name>"
    fi
}
```

**Usage examples:**
```bash
./uninstall.sh                  # Safe: keeps data, images
./uninstall.sh --remove-data    # Removes volumes (data loss!)
./uninstall.sh --all            # Complete cleanup
./uninstall.sh --force --all    # No prompts, remove everything
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
| `tsx` / `ts-node` not found in prod | Install `tsx` globally in prod OR use it only if TS has imports |
| `migrate deploy` with no migrations | Use `db push` if no `prisma/migrations/` directory |
| `tsc seed.ts` fails with imports | Use `tsx` (handles imports) or compile entire project |
| tsx can't find npm packages in monorepo | Create symlinks in Dockerfile (NODE_PATH doesn't work with tsx) |
| "prisma generate" error after finding client | Copy `.prisma/client` to root node_modules, not just workspace |
| Silent build failures with `\|\| true` | Remove `\|\| true` or add explicit error handling |
| Build OOMs on low-resource VPS | Add swap space; use `NODE_OPTIONS=--max-old-space-size=512` |
| Docker build freezes on 1GB VPS | Configure swap (2x RAM), limit concurrent processes |
| Dockerfile changed but old behavior | Docker layer caching; use `docker system prune -af` or `--no-cache` |
| certbot "No renewals attempted" | Override entrypoint: `--entrypoint "" certbot certbot certonly` |
| nginx "host not found in upstream" | Start API container before nginx; use `depends_on` with healthcheck |
| ENCRYPTION_KEY too short (32 chars) | Generate 64+ characters: `openssl rand -base64 48 \| tr -dc 'a-zA-Z0-9' \| head -c 64` |

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
- [ ] No `|| true` on critical commands (compilation, migrations)
- [ ] TypeScript scripts with imports use tsx (not tsc standalone)

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

### Low-Resource VPS (≤2GB RAM)
- [ ] Swap space configured (2x RAM recommended)
- [ ] `NODE_OPTIONS="--max-old-space-size=512"` set in Dockerfile
- [ ] Docker daemon optimized (log rotation, overlay2 storage)
- [ ] Removed `--no-cache` from docker build (leverage layer caching)
- [ ] Consider pre-building images on more powerful machine

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

---

## References & Resources

### Official Documentation
- [Docker Best Practices](https://docs.docker.com/build/building/best-practices/) — Official Docker build guide
- [Node.js Docker Best Practices](https://github.com/nodejs/docker-node/blob/main/docs/BestPractices.md) — Node.js specific patterns
- [npm Documentation](https://docs.npmjs.com/) — Package management reference
- [Alpine Linux Packages](https://pkgs.alpinelinux.org/packages) — Find required system deps

### Security Resources
- [OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html) — Container security
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker) — Industry security standards
- [Snyk Container Security](https://snyk.io/learn/container-security/) — Vulnerability scanning
- [Docker Scout](https://docs.docker.com/scout/) — Image analysis and CVE detection
- [Trivy](https://aquasecurity.github.io/trivy/) — Open-source vulnerability scanner

### npm Security
- [npm Audit Documentation](https://docs.npmjs.com/cli/v10/commands/npm-audit) — Vulnerability auditing
- [Snyk Advisor](https://snyk.io/advisor/) — Package health scores
- [Socket.dev](https://socket.dev/) — Supply chain security

### Build Optimization
- [Docker Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/) — Reduce image size
- [Docker Layer Caching](https://docs.docker.com/build/cache/) — Speed up builds
- [BuildKit](https://docs.docker.com/build/buildkit/) — Advanced build features

### Compliance & Standards
- [SOC 2 Container Requirements](https://www.vanta.com/resources/soc-2-compliance-checklist) — Audit requirements
- [HIPAA Technical Safeguards](https://www.hhs.gov/hipaa/for-professionals/security/laws-regulations/index.html) — Healthcare compliance
- [NIST Container Security Guide](https://csrc.nist.gov/publications/detail/sp/800-190/final) — SP 800-190

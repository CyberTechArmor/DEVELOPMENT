# Docker & Monorepo Codebase Review Prompt

> Use this prompt when reviewing a codebase for Docker, Prisma, and monorepo best practices.

---

## Prompt for Claude Code

```
Review this codebase against Docker and monorepo best practices. Check each category below and report findings with specific file locations and fixes.

## 1. Dockerfile Analysis

Check for these issues in Dockerfile(s):

### npx Usage (Critical)
- [ ] Search for `npx` in RUN commands
- Fix: Replace with `./node_modules/.bin/<command>` or npm scripts
- Why: npx downloads latest version, ignoring pinned versions

### Silent Failures
- [ ] Search for `|| true` on compilation, migration, or install commands
- Fix: Remove `|| true` or add explicit error handling with logging
- Why: Masks failures that only appear at runtime

### Multi-stage Build Efficiency
- [ ] Check if artifacts are regenerated after COPY --from
- Fix: Remove redundant RUN commands after copying built artifacts
- Why: Wastes build time regenerating what's already built

### Alpine Dependencies
- [ ] If using node:*-alpine, check for `openssl` installation
- Fix: Add `RUN apk add --no-cache openssl` for Prisma
- Why: Prisma requires libssl on Alpine

### Production Settings
- [ ] Verify `NODE_ENV=production` is set
- [ ] Verify `npm ci --omit=dev` is used (not `npm install`)
- [ ] Check source maps are disabled in production image

## 2. Package.json & Dependencies

### Runtime vs Dev Dependencies
- [ ] Check if `prisma` CLI is in dependencies (not devDependencies) if used in production
- [ ] Check if any devDependencies are called in Dockerfile or install scripts
- Fix: Move runtime CLIs to dependencies

### Monorepo Hoisting
- [ ] If using npm workspaces, check if shared deps are in root package.json
- [ ] Verify `@prisma/client` and `prisma` are hoisted to root
- Fix: Add to root package.json to force hoisting
- Command: `npm install --package-lock-only --ignore-scripts` after changes

### Version Pinning
- [ ] Check for exact versions on critical deps (Prisma, etc.)
- [ ] Look for `_versionNotes` or VERSIONS.md documenting pins

## 3. TypeScript in Production

### Seed Scripts with Imports
- [ ] Check if seed.ts imports from project files (../src/*, etc.)
- If yes: Must use tsx (tsc standalone won't work)
- [ ] Verify tsx is installed globally in production image
- [ ] Check Dockerfile creates symlinks for module resolution:
  ```dockerfile
  RUN mkdir -p /app/apps/api/node_modules/@prisma && \
      ln -s /app/node_modules/@prisma/client /app/apps/api/node_modules/@prisma/client
  ```

### NODE_PATH Warning
- [ ] If using NODE_PATH for tsx, flag it - tsx ignores NODE_PATH
- Fix: Use symlinks instead

## 4. Install/Uninstall Scripts

### Install Script
- [ ] Check for `--no-cache` flag (should be optional, not default)
- [ ] Verify hardware detection exists for low-resource VPS
- [ ] Check for `--help` output
- [ ] Verify `docker compose down -v` for fresh installs (credential sync)

### Uninstall Script
- [ ] Check for graduated cleanup levels (--remove-data, --remove-images, --all)
- [ ] Verify `--force` option for non-interactive mode
- [ ] Check completion message shows what was/wasn't removed

## 5. Shell Script Container Commands

### Path Issues
- [ ] Search for `docker compose run/exec` commands
- [ ] Verify they use full paths: `/app/node_modules/.bin/prisma`
- [ ] Verify correct working directory for Prisma: `cd /app/apps/api`

### Seed Script Command
- [ ] If monorepo with tsx, verify seed runs from root with symlinks:
  ```bash
  docker compose run --rm api sh -c "cd /app && tsx apps/api/prisma/seed.ts"
  ```

## 6. Low-Resource VPS (if applicable)

- [ ] Check for NODE_OPTIONS="--max-old-space-size=512" in Dockerfile
- [ ] Check install script has swap configuration
- [ ] Verify Docker cache is leveraged (no `--no-cache` by default)

## 7. Volume & Credential Management

- [ ] Install script cleans volumes before generating new credentials
- [ ] Update script does NOT use `-v` flag (preserves data)
- [ ] Check for credential mismatch risk documentation

---

## Report Format

For each issue found, report:

1. **File**: Path and line number
2. **Issue**: What's wrong
3. **Impact**: What breaks
4. **Fix**: Specific code change

Example:
```
**File**: docker/Dockerfile:47
**Issue**: Using `npx prisma generate` instead of local binary
**Impact**: Downloads Prisma 7.x instead of pinned 5.x, breaks schema
**Fix**: Change to `./node_modules/.bin/prisma generate`
```

---

## Quick Verification Commands

After fixes, verify with:

```bash
# Check no npx in Dockerfile
grep -n "npx" Dockerfile

# Check silent failures
grep -n "|| true" Dockerfile

# Verify production environment in container
docker run --rm <image> printenv NODE_ENV

# Check module locations
docker run --rm <image> ls -la /app/node_modules/.bin/prisma
docker run --rm <image> ls -la /app/apps/api/node_modules/@prisma/

# Check no devDependencies leaked
docker run --rm <image> npm ls --omit=dev 2>&1 | grep -E "typescript|eslint|vitest"
```
```

---

## Reference

This prompt is based on learnings documented in:
- [Docker Dependency Management](../guides/docker-dependency-management.md)
- [Prisma & TypeScript Workflows](../guides/prisma-typescript-workflows.md)

Key sections:
- Section 1: npx version resolution
- Section 4: Shell script container commands
- Section 6: npm workspaces hoisting
- Section 9: TypeScript scripts in production (tsx + symlinks)
- Section 10: Silent failures with || true
- Section 11: Low-resource VPS optimization
- Section 12: Install/uninstall script patterns

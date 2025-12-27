# Prisma & TypeScript Workflows

## Correct Terminology

When working with Prisma and TypeScript, use these precise terms:

| Common (Informal) Phrase | Correct Technical Term | Command |
|--------------------------|------------------------|---------|
| "Aligning Prisma/TypeScript" | **Generating Prisma Client** | `npx prisma generate` |
| "Seeding the database" | **Database Seeding** | `npx prisma db seed` |
| "Deploying the schema" | **Running Migrations** | `npx prisma migrate deploy` (prod) |
| "Updating the schema" | **Running Migrations** | `npx prisma migrate dev` (dev) |
| "Pushing schema directly" | **Schema Push** | `npx prisma db push` (no migrations) |

---

## Understanding the Workflow

### 1. Generating Prisma Client (`npx prisma generate`)

Creates TypeScript types from your `schema.prisma` file.

- Generated types live in `node_modules/.prisma/client`
- **Must be run after any schema changes** for types to stay in sync
- Without this step, your TypeScript code will reference outdated types

```bash
npx prisma generate
```

### 2. Database Seeding (`npx prisma db seed`)

Populates the database with initial or test data.

- Defined in your `prisma/seed.ts` (or `.js`) file
- Configured in `package.json` under `prisma.seed`

```bash
npx prisma db seed
```

### 3. Running Migrations

Applies schema changes to the database.

**Development:**
```bash
npx prisma migrate dev
```
- Creates migration files
- Applies them to your dev database
- Regenerates Prisma Client automatically

**Production:**
```bash
npx prisma migrate deploy
```
- Applies existing migration files only
- Does NOT create new migrations
- Safe for CI/CD pipelines

### 4. Schema Push vs Migrations (`db push` vs `migrate deploy`)

Two different approaches to applying schema changes:

| Command | Use Case | Creates Migration Files? | Safe for Production? |
|---------|----------|--------------------------|----------------------|
| `prisma db push` | Prototyping, no migrations needed | ❌ No | ⚠️ Use with caution |
| `prisma migrate deploy` | Production with migration history | ✅ Uses existing | ✅ Yes |

**When to use `db push`:**
```bash
npx prisma db push --skip-generate
```
- Project doesn't use migration files (no `prisma/migrations/` directory)
- Rapid prototyping or initial setup
- Schema-first development without migration history
- Docker/containerized deployments where schema IS the source of truth

**When to use `migrate deploy`:**
```bash
npx prisma migrate deploy
```
- Project has `prisma/migrations/` directory with migration files
- Need audit trail of schema changes
- Team collaboration requiring consistent migrations
- Rollback capability needed

**Common error when using wrong command:**
```
Error: P3005
The database schema is not empty. Read more about how to baseline
an existing production database: https://pris.ly/d/migrate-baseline
```

This means you're trying to use `migrate deploy` but there are no migration files. Either:
1. Create a baseline migration: `npx prisma migrate dev --name init`
2. Or use `db push` if you don't need migrations

### 5. Seeding in Production

The default `prisma db seed` command runs the seed script specified in `package.json`:

```json
{
  "prisma": {
    "seed": "tsx prisma/seed.ts"
  }
}
```

**Problem:** `tsx` is typically a devDependency and won't be in production containers.

**Solution 1: Compile seed.ts during Docker build**

```dockerfile
# In Dockerfile, after npm run build:
RUN cd apps/api && ../../node_modules/.bin/tsc prisma/seed.ts \
    --outDir prisma \
    --esModuleInterop \
    --skipLibCheck \
    --resolveJsonModule || true
```

Then run with Node directly:
```bash
# Instead of: npx prisma db seed
node prisma/seed.js
```

**Solution 2: Use a JavaScript seed file**

Write `prisma/seed.js` instead of `seed.ts`:
```javascript
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

async function main() {
  // Seed logic here
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

**Solution 3: Include tsx in production dependencies**

Only if seeding is truly needed at runtime (usually not recommended):
```json
{
  "dependencies": {
    "tsx": "^4.7.0"  // Move from devDependencies
  }
}
```

---

## Avoiding Issues with AI Coding

### The Core Problem

AI coding assistants may assume model names without verifying against the actual schema. For example, assuming a model is named `BookingAttendee` when the schema actually defines it as `Attendee`.

### Best Practices

#### 1. Always verify generated types after schema changes

```bash
npx prisma generate
```

Then verify your model names:

```bash
grep "^model " prisma/schema.prisma
```

#### 2. Use Prisma's type inference instead of direct model imports

Direct imports can break when schema changes:

```typescript
// Fragile - can break if model is renamed or doesn't exist
import { Attendee } from '@prisma/client';
```

**Preferred approach** - use Prisma's inference utilities:

```typescript
// Robust - automatically reflects your actual schema
import { Prisma } from '@prisma/client';

type BookingWithRelations = Prisma.BookingGetPayload<{
  include: { attendees: true; host: true; eventType: true }
}>;
```

This pattern:
- Automatically updates when schema changes
- Catches errors at compile time
- Documents the expected shape of your data

#### 3. Run type-check before committing

```bash
npm run build
# or for type-checking only:
npx tsc --noEmit
```

#### 4. Add a pre-commit hook

In `package.json`:

```json
{
  "scripts": {
    "precommit": "npx prisma generate && npx tsc --noEmit"
  }
}
```

Or use [Husky](https://typicode.github.io/husky/) for git hooks:

```bash
npx husky add .husky/pre-commit "npx prisma generate && npx tsc --noEmit"
```

#### 5. Provide context to AI coding assistants

When starting an AI coding session involving Prisma:

1. Share the Prisma schema directly, or provide output of:
   ```bash
   grep "^model " prisma/schema.prisma
   ```

2. Run the build locally before pushing to catch type errors early

3. If the AI generates imports for model types, verify them against your schema

---

## Quick Reference Commands

```bash
# View all model names in your schema
grep "^model " prisma/schema.prisma

# Regenerate client after schema changes
npx prisma generate

# Create and apply new migration (development)
npx prisma migrate dev --name your_migration_name

# Apply existing migrations (production/CI)
npx prisma migrate deploy

# Push schema directly without migrations (prototyping/Docker)
npx prisma db push --skip-generate

# Seed the database (development with tsx)
npx prisma db seed

# Seed in production (compiled JavaScript)
node prisma/seed.js

# Reset database (drops all data!)
npx prisma migrate reset

# Open Prisma Studio (visual database browser)
npx prisma studio

# Validate schema syntax
npx prisma validate

# Format schema file
npx prisma format

# Check if migrations directory exists
ls prisma/migrations/ 2>/dev/null || echo "No migrations - use db push"
```

---

## Common Mistakes Summary

| Mistake | Solution |
|---------|----------|
| Assuming model names without checking | Run `grep "^model " prisma/schema.prisma` |
| Forgetting to regenerate client | Always run `npx prisma generate` after schema changes |
| Importing model types directly | Use `Prisma.ModelGetPayload<{...}>` pattern |
| Pushing without type-checking | Add pre-commit hook with `tsc --noEmit` |
| Confusing `migrate dev` vs `migrate deploy` | `dev` creates migrations; `deploy` applies existing ones |
| Using `migrate deploy` with no migrations | Check for `prisma/migrations/`; use `db push` if missing |
| `prisma db seed` fails in production | Compile `seed.ts` to JS during build; run with `node` |
| `tsx` not found in container | `tsx` is devDependency; compile TypeScript or use JS seed |

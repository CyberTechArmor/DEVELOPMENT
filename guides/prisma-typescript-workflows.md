# Prisma & TypeScript Workflows

## Correct Terminology

When working with Prisma and TypeScript, use these precise terms:

| Common (Informal) Phrase | Correct Technical Term | Command |
|--------------------------|------------------------|---------|
| "Aligning Prisma/TypeScript" | **Generating Prisma Client** | `npx prisma generate` |
| "Seeding the database" | **Database Seeding** | `npx prisma db seed` |
| "Deploying the schema" | **Running Migrations** | `npx prisma migrate deploy` (prod) |
| "Updating the schema" | **Running Migrations** | `npx prisma migrate dev` (dev) |

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

# Seed the database
npx prisma db seed

# Reset database (drops all data!)
npx prisma migrate reset

# Open Prisma Studio (visual database browser)
npx prisma studio

# Validate schema syntax
npx prisma validate

# Format schema file
npx prisma format
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

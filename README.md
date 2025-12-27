# Development Reference Guide

A repository for learnings, best practices, and industry standards accumulated during software development.

## Contents

| Topic | Description |
|-------|-------------|
| [Prisma & TypeScript Workflows](./guides/prisma-typescript-workflows.md) | Schema generation, migrations, seeding, and avoiding common pitfalls with AI coding |
| [Docker Dependency Management](./guides/docker-dependency-management.md) | Version pinning, avoiding npx pitfalls, eliminating redundant build steps, and build auditing |
| [Security & Compliance](./guides/security-compliance.md) | SOC 2, HIPAA, PCI DSS, GDPR, ISO 27001 — unified compliance mapping, container security, encryption |
| [Technology Stack](./guides/technology-stack.md) | Fastify, React, TypeScript, PostgreSQL, Redis — stack recommendations with compliance considerations |

## Review Prompts

Prompts for AI-assisted code review:

| Prompt | Use Case |
|--------|----------|
| [Docker & Monorepo Review](./prompts/docker-monorepo-review.md) | Review Dockerfile, install scripts, and monorepo patterns against best practices |

## Purpose

This guide serves as a living document to capture:

- **Correct terminology** for common development tasks
- **Best practices** learned through real-world experience
- **Common pitfalls** and how to avoid them
- **Industry standards** for tooling and workflows

## Contributing

Each guide should include:
1. Correct terminology with explanations
2. Practical examples and commands
3. Common mistakes and how to avoid them
4. Pre-commit hooks or automation where applicable

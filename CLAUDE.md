# Engineering Playbook

Reusable engineering standards for full-stack projects. Read relevant files before starting implementation.

**Rule:** When a project's own CLAUDE.md conflicts with this playbook, the project wins.

## Quick Reference by Task

| Task | Read these files |
|---|---|
| New API endpoint | `architecture/api-design.md`, `security/server-side-validation.md`, `patterns/error-handling.md` |
| Auth / admin gate | `architecture/auth-and-sessions.md`, `security/auth-security.md` |
| Payment flow | `security/payment-security.md`, `patterns/pricing-and-money.md` |
| Backend tests | `testing/backend-testing.md`, `testing/coverage-policy.md` |
| Frontend tests | `testing/frontend-testing.md`, `testing/coverage-policy.md` |
| Frontend feature | `architecture/frontend-patterns.md`, `code-quality/typescript-standards.md` |
| New deployment | `devops/ci-pipeline.md`, `devops/deployment.md`, `devops/docker-and-compose.md` |
| PR review | `code-quality/review-checklist.md` |
| Error handling | `patterns/error-handling.md` |
| Money / pricing | `patterns/pricing-and-money.md`, `code-quality/python-standards.md` |
| Docker setup | `devops/docker-and-compose.md`, `devops/environment-config.md` |
| CI/CD pipeline | `devops/ci-pipeline.md` |
| Project structure | `architecture/monorepo-structure.md` |

## Conventions

- Files are self-contained and readable in isolation
- Each file uses headers, checklists, and short code snippets
- Patterns are extracted from real production projects
- "Why" explanations are included for non-obvious rules
- Anti-patterns are called out where relevant

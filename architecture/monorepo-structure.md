# Monorepo Structure

## Recommended Layout

```
project-root/
├── apps/
│   ├── web/          # Frontend (Next.js / React)
│   └── api/          # Backend (FastAPI / Node)
├── supabase/         # Database config + migrations
├── nginx/            # Reverse proxy config
├── .github/workflows/  # CI/CD
├── docker-compose.yml
├── CLAUDE.md
└── README.md
```

## Principles

- **Each app is independently buildable and testable** — has its own package.json / requirements.txt
- **Shared config at root** — docker-compose, CI workflows, project-level CLAUDE.md
- **No shared code packages unless 3+ consumers** — premature abstractions hurt more than duplication
- **Database migrations live in `supabase/`** — not inside any app

## Package Boundaries

- `apps/web/` should only import from its own `lib/`, `components/`, `app/` directories
- `apps/api/` should only import from its own modules
- Apps communicate via HTTP API — never import code across apps
- Shared types are duplicated (TypeScript interfaces in web, Pydantic models in api)

## Why Not a Shared Package?

Shared packages create coupling, versioning headaches, and build complexity. Until you have 3+ consumers of the same logic, keep types and utilities in the app that uses them.

## Anti-patterns

- Importing backend code in frontend or vice versa
- Single package.json / requirements.txt for the whole monorepo
- Shared `lib/` folder at root that both apps import from
- Database access from the frontend app directly (bypass the API)

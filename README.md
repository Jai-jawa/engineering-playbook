# Engineering Playbook

Reusable engineering standards for full-stack web applications. Extracted from real production projects.

## What This Is

A structured collection of markdown files covering architecture, code quality, testing, security, DevOps, and common patterns. Designed to be read by both humans and Claude Code for consistent engineering practices across projects.

## Structure

```
architecture/     → Project layout, API design, auth, data flow, frontend
code-quality/     → Python & TypeScript standards, PR review checklist
testing/          → Backend (Pytest), frontend (Vitest), coverage policy
security/         → Validation, auth, payments, infrastructure
devops/           → Docker, CI/CD, deployment, environment config
patterns/         → Error handling, pricing/money, email
```

## How to Use

### With Claude Code
```bash
# Option A: Add as additional context directory
claude --add-dir ~/engineering-playbook

# Option B: Symlink into your project
ln -s ~/engineering-playbook .playbook

# Option C: Git submodule
git submodule add https://github.com/yourusername/engineering-playbook.git .playbook
```

Then reference in your project's `CLAUDE.md`:
```markdown
Read `.playbook/CLAUDE.md` for baseline engineering standards.
Project-specific rules in this file take precedence.
```

### For Humans
Browse the files directly. Each file is self-contained with headers, checklists, code snippets, and anti-patterns.

## Tech Stack Coverage

These patterns are written for:
- **Backend:** Python 3.10+, FastAPI, SQLAlchemy async, Pydantic v2
- **Frontend:** Next.js 14, React 18, TypeScript
- **Database:** PostgreSQL (Supabase)
- **Infrastructure:** Docker, Nginx, GitHub Actions, DigitalOcean

The principles (server-side validation, Decimal math, test patterns) apply to any stack.

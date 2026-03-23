# ADR-002: Use FastAPI for HTTP API Server

## Status
Accepted

## Context
Need an HTTP API framework for the business logic layer. Options:
1. **FastAPI** (Python, async)
2. **Django REST Framework** (Python, sync-first)
3. **Express.js** (Node.js)
4. **Go with Gin/Echo** (Go)

## Decision
Use FastAPI (Python) for the HTTP API server.

## Rationale
- Async-native — critical for high-throughput SMS processing and non-blocking I/O
- Pydantic models provide request/response validation with strong typing
- Auto-generated OpenAPI docs (Swagger/ReDoc) — reduces documentation maintenance
- Python ecosystem has good SMPP libraries if we ever need to bypass Jasmin
- Faster development speed vs Go for a small team
- Better async support than Django REST Framework

## Consequences
- **Good:** Strong typing via Pydantic, auto-docs, fast development
- **Good:** Async supports high concurrency without threading complexity
- **Bad:** Python is slower than Go for CPU-bound work — mitigated by SMS being I/O-bound
- **Bad:** Python's GIL limits true parallelism — mitigated by async + multiple workers
- **Risk:** If we need >1000 TPS, may need to optimize or consider Go for specific hot paths

# Auth Security

## JWT Verification

### Algorithm Restriction (CRITICAL)

```python
ALLOWED_ALGORITHMS = {"HS256", "ES256"}

unverified_header = jwt.get_unverified_header(token)
alg = unverified_header.get("alg")
if alg not in ALLOWED_ALGORITHMS:
    raise HTTPException(401, f"Unsupported algorithm: {alg}")
```

**Why:** Prevents alg:none attacks where an attacker sends an unsigned token with `{"alg": "none"}`. The server would skip verification entirely.

### JWKS Caching

```python
class JWKSCache:
    def __init__(self, ttl_seconds=3600):
        self._key = None
        self._fetched_at = 0
        self._lock = asyncio.Lock()
        self._ttl = ttl_seconds
        self._backoff_until = 0

    async def get_key(self, kid: str) -> jwt.PyJWK:
        now = time.monotonic()
        if self._key and (now - self._fetched_at) < self._ttl:
            return self._key

        async with self._lock:
            # Double-check after acquiring lock (another task may have refreshed)
            if self._key and (now - self._fetched_at) < self._ttl:
                return self._key

            if now < self._backoff_until:
                if self._key:
                    return self._key  # Stale cache > no cache
                raise HTTPException(503, "JWKS unavailable")

            try:
                self._key = await fetch_jwks(kid)
                self._fetched_at = now
            except Exception:
                self._backoff_until = now + 30  # 30s backoff
                if self._key:
                    return self._key
                raise
        return self._key
```

**Key patterns:**
- 1-hour TTL with monotonic clock (not wall clock)
- `asyncio.Lock` prevents thundering herd on concurrent requests
- Double-check pattern: re-check cache after acquiring lock
- Stale cache fallback on fetch failure
- 30-second backoff on repeated failures

### Verification Flow

1. Extract Bearer token from Authorization header
2. Read unverified header → get algorithm
3. Reject if algorithm not in whitelist
4. For ES256: fetch JWKS public key by `kid`, verify
5. For HS256: verify with shared secret
6. Always set `audience="authenticated"`

## Admin Authorization

```python
async def require_admin(request: Request) -> dict:
    claims = await verify_jwt(request)
    if not claims.get("app_metadata", {}).get("is_admin"):
        raise HTTPException(403, "Admin access required")
    return claims
```

- Admin role from JWT `app_metadata.is_admin` — **no database calls**
- Returns 401 for invalid token, 403 for non-admin
- Dev mode: any authenticated user is admin (controlled by APP_ENV)

## Rules

- Middleware returns a claims dict, never a user model
- Endpoint layer does DB lookups if user data is needed
- Use `Depends(require_admin)` for admin routes
- Reusable `httpx.AsyncClient` for JWKS fetches (close on shutdown)

## Anti-patterns

- Accepting any JWT algorithm (alg:none attack)
- Fetching JWKS on every request (thundering herd, latency)
- DB queries in auth middleware (unnecessary load, coupling)
- Storing admin flag in client-accessible storage
- Using wall clock (`time.time()`) for TTL (affected by system clock changes)

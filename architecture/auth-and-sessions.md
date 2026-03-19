# Auth & Sessions

## Auth Flow (Supabase SSR)

### Frontend
1. Supabase JS SDK handles sign-up, sign-in, token refresh
2. `AuthProvider` context listens to `supabase.auth.onAuthStateChange()`
3. Middleware refreshes session on every request (server-side)
4. JWT stored in HTTP-only cookies (Supabase SSR handles this)

### Backend
1. Extract Bearer token from Authorization header
2. Validate JWT (see `security/auth-security.md` for details)
3. Return claims dict: `{user_id, email, role, is_admin}`
4. Route handler uses claims — no DB lookup for auth

### Admin Routes
```python
@router.get("/admin/orders")
async def list_orders(admin: dict = Depends(require_admin)):
    # admin["user_id"], admin["email"], admin["is_admin"] available
```

- `require_admin` dependency validates JWT + checks `app_metadata.is_admin`
- Returns 401 for missing/invalid token, 403 for non-admin
- Dev mode shortcut: any authenticated user is admin

## Guest vs. Authenticated Sessions

### Guest Flow
1. Generate UUID session_id, store in localStorage
2. All cart operations use `session_id` parameter
3. No auth required for browsing or cart

### Authenticated Flow
1. User signs in via Supabase
2. Cart operations use `user_id` from JWT
3. `session_id` still sent as fallback

### Cart Merge on Login
```python
# On login, merge guest cart items to authenticated user
POST /cart/merge
Body: { session_id: "guest-uuid" }
Auth: Bearer token (provides user_id)

# Backend: UPDATE cart_items SET user_id = :uid WHERE session_id = :sid AND user_id IS NULL
```

**Critical:** Merge failure must not block login — catch and log.

## Anti-patterns

- Storing JWT in localStorage (use HTTP-only cookies)
- DB queries in auth middleware (use JWT claims only)
- Requiring auth for browsing/cart (guest experience matters)
- Blocking login flow on non-critical operations (merge, analytics)

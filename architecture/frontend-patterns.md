# Frontend Patterns (Next.js)

## Project Layout

```
apps/web/
├── app/                 # Next.js App Router (pages, layouts, routes)
│   ├── layout.tsx       # Root layout with providers
│   ├── page.tsx         # Home page
│   └── [slug]/          # Dynamic routes
├── components/          # Reusable UI components
├── lib/
│   ├── api.ts           # Centralized API client with typed interfaces
│   ├── auth-context.tsx # Auth state provider
│   ├── cart-context.tsx # Cart state provider
│   ├── supabase/        # Supabase client (browser, server, middleware)
│   └── utils.ts         # Shared utilities
└── public/              # Static assets
```

## Centralized API Client

All backend calls go through a single typed API client:

```typescript
// lib/api.ts
const API_URL = getApiUrl(); // from NEXT_PUBLIC_API_URL

export interface ProductConfig {
  slug: string;
  name: string;
  formula_type: string;
  option_groups: OptionGroup[];
}

export async function fetchProduct(slug: string): Promise<ProductConfig> {
  const res = await fetch(`${API_URL}/products/${slug}`);
  if (!res.ok) throw new Error(`Failed to fetch product: ${res.status}`);
  return res.json();
}
```

**Why:** Single source of truth for API types, base URL, and error handling.

## Context Providers

### Auth Context
- Wraps Supabase `onAuthStateChange` listener
- Provides: `user`, `isLoading`, `signIn()`, `signOut()`
- Triggers cart merge on login (non-blocking)

### Cart Context
- Manages guest (session_id) and authenticated (user_id) carts
- Provides: `items`, `addToCart()`, `updateQuantity()`, `removeItem()`
- Stores session_id in localStorage for guest persistence

### Provider hierarchy
```tsx
<AuthProvider>
  <CartProvider>
    {children}
  </CartProvider>
</AuthProvider>
```

Cart depends on Auth (needs user_id), so Auth wraps Cart.

## State Management Rules

- Use React Context for global app state (auth, cart)
- Use component state for UI state (modals, form inputs)
- No Redux/Zustand unless 5+ contexts needed
- Server state (products, orders) fetched on demand, not cached globally

## Error Handling in UI

```typescript
try {
  const data = await fetchCart(sid, user?.id);
  setItems(data.items);
} catch (err) {
  console.error("Failed to fetch cart:", err);
  // Show user-friendly error, don't crash
} finally {
  setIsLoading(false);
}
```

- Always set loading to false in `finally`
- Non-critical failures (merge, analytics) — catch and continue
- Critical failures (checkout, payment) — show error to user

## Anti-patterns

- Fetching from backend in multiple places without the API client
- Storing server-calculated values in client state (prices, totals)
- Blocking the auth flow on non-critical operations
- Using `useEffect` for data that should be server-rendered

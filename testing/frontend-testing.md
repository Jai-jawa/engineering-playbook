# Frontend Testing (Vitest + Testing Library)

## Configuration

### vitest.config.ts
```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./tests/setup.ts"],
  },
});
```

### Package Dependencies
```json
{
  "devDependencies": {
    "vitest": "^1.x",
    "@testing-library/react": "^14.x",
    "@testing-library/jest-dom": "^6.x",
    "jsdom": "^24.x"
  }
}
```

## Test Patterns

### Component Testing

```typescript
import { render, screen, fireEvent } from "@testing-library/react";
import { ProductCard } from "@/components/ProductCard";

test("displays product name and price", () => {
  render(<ProductCard product={mockProduct} onSelect={vi.fn()} />);
  expect(screen.getByText("Business Cards")).toBeInTheDocument();
  expect(screen.getByText("₹125.00")).toBeInTheDocument();
});

test("calls onSelect when clicked", async () => {
  const onSelect = vi.fn();
  render(<ProductCard product={mockProduct} onSelect={onSelect} />);
  await fireEvent.click(screen.getByRole("button"));
  expect(onSelect).toHaveBeenCalledWith("business-cards");
});
```

### Testing with Context Providers

```typescript
function renderWithProviders(ui: React.ReactElement) {
  return render(
    <AuthProvider>
      <CartProvider>
        {ui}
      </CartProvider>
    </AuthProvider>
  );
}

test("shows cart count", () => {
  renderWithProviders(<CartIcon />);
  expect(screen.getByText("3")).toBeInTheDocument();
});
```

### Mocking API Calls

```typescript
import { vi } from "vitest";
import * as api from "@/lib/api";

vi.mock("@/lib/api");

test("fetches products on mount", async () => {
  vi.mocked(api.fetchProducts).mockResolvedValue([mockProduct]);
  render(<ProductList />);
  expect(await screen.findByText("Business Cards")).toBeInTheDocument();
});
```

### Testing Async State

```typescript
test("shows loading then content", async () => {
  render(<ProductList />);
  expect(screen.getByText("Loading...")).toBeInTheDocument();
  expect(await screen.findByText("Business Cards")).toBeInTheDocument();
  expect(screen.queryByText("Loading...")).not.toBeInTheDocument();
});
```

## What to Test

- [ ] Components render correctly with given props
- [ ] User interactions trigger correct callbacks
- [ ] Loading and error states display properly
- [ ] Conditional rendering based on auth state
- [ ] Form validation and submission

## What NOT to Test

- Third-party library internals (Supabase, Next.js routing)
- CSS styling (use visual regression tools if needed)
- Implementation details (state variables, internal methods)

## Running Tests

```bash
npm run test        # Single run
npx vitest          # Watch mode
npx vitest --coverage  # With coverage
```

## Anti-patterns

- Testing implementation details (`component.state.value`)
- Using `container.querySelector` instead of Testing Library queries
- Not awaiting async updates (`findBy*` queries)
- Testing that a mock was called without testing the UI effect

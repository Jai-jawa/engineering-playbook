# TypeScript Standards

## Strict Mode

Always enable in `tsconfig.json`:
```json
{
  "compilerOptions": {
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "paths": { "@/*": ["./*"] }
  }
}
```

- `strict: true` enables all strict checks
- Path aliases (`@/`) for clean imports
- Run `tsc --noEmit` in CI

## Typed API Interfaces

Define interfaces for all API responses in a central `lib/api.ts`:

```typescript
export interface ProductConfig {
  id: number;
  slug: string;
  name: string;
  formula_type: "perSheet" | "multiplicative" | "rateDivisor" | "area";
  option_groups: OptionGroup[];
}

export interface OptionGroup {
  key: string;
  display_label: string;
  input_type: "select" | "input" | "dimension";
  options: OptionValue[];
}
```

- Use union types for known string values (not plain `string`)
- Match backend Pydantic models 1:1
- Update both when adding/changing fields

## ESLint

Use Next.js default config + strict rules:
```json
{
  "extends": "next/core-web-vitals"
}
```

Run `next lint` in CI.

## Component Patterns

### Props Typing
```typescript
interface ProductCardProps {
  product: ProductConfig;
  onSelect: (slug: string) => void;
}

export function ProductCard({ product, onSelect }: ProductCardProps) { ... }
```

- Name props interfaces `{ComponentName}Props`
- Destructure props in function signature
- Use named exports (not default exports) for components

### Event Handlers
```typescript
// Typed event handlers
const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault();
  // ...
};
```

## State Typing

```typescript
const [items, setItems] = useState<CartItem[]>([]);
const [error, setError] = useState<string | null>(null);
```

- Always provide generic type to `useState` when initial value doesn't infer correctly
- Use `null` not `undefined` for "no value" states

## Anti-patterns

- Using `any` type (use `unknown` if type is truly unknown)
- Disabling TypeScript checks with `@ts-ignore` (use `@ts-expect-error` with explanation if unavoidable)
- Plain `string` for values with known options (use union types)
- Mixing default and named exports

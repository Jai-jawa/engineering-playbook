# PR Review Checklist

## Before Submitting

- [ ] Code compiles / type-checks with no errors
- [ ] Linter passes (ruff check / eslint)
- [ ] Tests pass locally
- [ ] No console.log / print statements left in
- [ ] No hardcoded secrets, URLs, or environment-specific values
- [ ] No `any` types in TypeScript

## Security

- [ ] Server-side validation for all user inputs
- [ ] No client-provided prices accepted by server
- [ ] Auth checks on protected routes
- [ ] No SQL injection (use parameterized queries / ORM)
- [ ] No XSS (React escapes by default; watch for `dangerouslySetInnerHTML`)
- [ ] Secrets in env vars, not in code

## API Changes

- [ ] Request models validated with Pydantic
- [ ] Response model defined (`response_model=`)
- [ ] Correct HTTP status codes (201 for create, 204 for delete)
- [ ] Error responses are informative but don't leak internals
- [ ] Matching TypeScript interfaces updated in frontend

## Database

- [ ] Migrations included if schema changed
- [ ] Indexes on frequently queried columns
- [ ] Foreign key constraints in place
- [ ] No N+1 queries (use joins or batch fetches)
- [ ] Decimal for money columns (not float)

## Frontend

- [ ] Loading states handled (`isLoading`)
- [ ] Error states handled (try/catch with user-friendly message)
- [ ] Responsive on mobile
- [ ] No unnecessary re-renders (check dependency arrays)
- [ ] Accessible (semantic HTML, alt text, keyboard navigation)

## Testing

- [ ] Happy path tested
- [ ] Edge cases covered (empty, null, boundary values)
- [ ] Auth scenarios tested (unauthenticated, wrong role)
- [ ] New code meets coverage threshold

## Performance

- [ ] No unnecessary network requests
- [ ] Large lists virtualized or paginated
- [ ] Images optimized (Next.js `<Image>` component)
- [ ] No blocking operations in async handlers

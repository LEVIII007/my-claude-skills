# Go Coding Standards

Apply when working on `**/*.go` files.

## Error Handling

- Check errors immediately — never ignore or discard
- Wrap with context: `fmt.Errorf("operation failed: %w", err)`
- Use `errors.Is` / `errors.As` for inspection — never string-compare error messages
- Define sentinel errors for expected domain failures: `var ErrNotFound = errors.New("not found")`
- Log errors only at system boundaries — never log AND return the same error
- Unexpected errors must propagate upward

## API Design & Visibility

- Only export identifiers required outside the package — minimize the public API surface
- Struct fields should be unexported unless external access is required
- Prefer constructor functions over direct struct initialization
- Avoid leaking implementation details through exported types
- Interfaces should not be exported unless required by external consumers
- Use `internal/` packages for non-public components
- Avoid circular dependencies across packages

## Struct & Interface Design

- Favor composition over inheritance
- Keep interfaces small and behavior-focused
- Define interfaces in the consuming package, not the implementing package
- Accept interfaces, return concrete types
- Avoid empty interfaces (`any`) unless absolutely required
- Avoid deep struct embedding chains
- Zero value of every struct must be valid and usable
- Structs containing synchronization primitives must not be copied

## Dependency Management

- All dependencies must be injected explicitly — no hidden dependencies
- No global mutable state — no package-level mutable variables
- Avoid tight coupling between layers
- Constructors must validate required dependencies

## Context & Lifecycle

- `context.Context` must be the first parameter of all request-scoped functions
- Never store context in struct fields
- Context cancellation must always be respected; deadlines must propagate downstream
- Never use `context.Background()` in request-scoped code paths

## Concurrency

- Do not communicate by sharing memory; share memory by communicating — prefer channels over mutexes where possible
- Goroutines must have a clearly defined lifecycle with a deterministic shutdown path
- All long-running operations must accept and respect `context.Context`
- No unbounded goroutine spawning — bound concurrency using worker pools or semaphores
- Only the sender may close a channel — receivers must never close
- Synchronization primitives must not be copied
- Avoid deadlocks by designing clear ownership models

## Worker Pools

- Worker pools must have bounded concurrency
- Task queues must not grow unbounded
- Workers must exit cleanly when context is cancelled
- Backpressure must be applied under load

## Receiver Semantics

- Use pointer receivers when mutation occurs, when structs are large, or when structs contain sync primitives
- Use value receivers only for small, immutable types
- Receiver type must be consistent across all methods of a type

## Logging

- Logging must occur only at system boundaries
- Business logic must not log unless required for observability
- Errors must not be logged multiple times
- Logging must be structured
- Sensitive information must never be logged

## Performance & Memory

- Avoid unnecessary allocations and copying of large structs
- Measure before optimizing — avoid premature optimization
- Critical paths must be benchmarked
- Avoid reflection unless required

## Testability

- All public functions must be testable and designed for deterministic testing
- Time-dependent logic must be injectable
- External systems must be abstracted behind interfaces — no direct filesystem, network, or time dependencies without abstraction
- When adding or modifying exported or package behavior, create or update the corresponding `*_test.go`; follow `testing.md` (table-driven tests, edge cases, error validation)

## Code Quality

- Code must pass `go vet`, static analysis, and race detector
- Code must not contain unused code or commented-out blocks
- Code must follow `gofmt` formatting
- Code must not introduce breaking changes without versioning

## Failure Design

- Systems must fail fast on programmer errors
- Systems must degrade gracefully on runtime failures
- Partial failures must be handled explicitly
- Retry logic must be bounded — use exponential backoff when retrying external systems

## Naming Conventions

### Identifiers
- Use `camelCase` for unexported identifiers, `PascalCase` for exported — never `snake_case`, `SCREAMING_SNAKE_CASE`, or other variants
- Acronyms and initialisms must use consistent case within the identifier: `userID` not `userId`, `HTTPClient` not `HttpClient`, `parseURL` not `parseUrl`
- Do not include the type in identifier names: `count` not `intCount`, `results` not `resultSlice` — exception: when distinguishing an original value from a type-converted copy (e.g. `userIDStr := strconv.Itoa(userID)`)
- Do not shadow Go builtins: avoid names like `int`, `bool`, `any`, `len`, `min`, `max`, `clear`
- Avoid names that clash with imported stdlib package names (e.g. don't name a variable `url` or `log` if those packages are imported)
- Use ASCII letters only in identifiers
- Identifier length should reflect scope: short names (`i`, `p`, `n`) are acceptable for narrow, tight scopes; descriptive names are required for wider scope or exported identifiers

### Packages
- Package names must be lowercase ASCII letters and numbers only — no underscores, no camelCase
- Prefer short, single-word nouns; concatenate multi-word names with no separator (`ordermanager` not `order_manager` or `orderManager`)
- Avoid catch-all names: `utils`, `helpers`, `common`, `types`, `interfaces` — use a focused, specific name instead
- Avoid names that clash with commonly-used stdlib packages (e.g. `url`, `log`, `time`)

### Files
- File names must be lowercase, ideally a single word (e.g. `server.go`, `handler.go`)
- For multi-word file names, pick one style — either underscore-separated (`routing_index.go`) or concatenated (`routingindex.go`) — and apply it consistently within the codebase
- Reserve the `_` separator for Go special suffixes only: `_test.go`, `_linux.go`, `_amd64.go` etc.
- Do not prefix file names with `.` or `_` (invisible to Go tooling)

### Receivers
- Receiver names must be 1–3 characters, typically an abbreviation of the type name (`c` for `*Client`, `hs` for `*HighScore`)
- Never use `this`, `self`, or `me` as receiver names
- The same receiver name must be used consistently across all methods of a type

### Getters and Setters
- Getter methods must not use a `Get` prefix — use the field or concept name directly (`Address()` not `GetAddress()`)
- Setter methods must use a `Set` prefix (`SetAddress()`)

### Interfaces
- Single-method interfaces should be named by the method name plus an `-er` suffix (`Reader`, `Authorizer`, `Stringer`)
- Do not use `Interface` as a name suffix (`UserInterface` → use a role-based name instead)

### Avoiding Chatter
- Exported function and type names must not repeat the package name: `customer.New()` not `customer.NewCustomer()`, `customer.Address` not `customer.CustomerAddress`
- Method names should not repeat the receiver type name: `(t *Token) Validate()` not `(t *Token) ValidateToken()`

## Tests

See `standards/go/testing.md` for test patterns, templates, and execution commands.

## Linting

See `standards/go/linters.md` for linting tools and commands.

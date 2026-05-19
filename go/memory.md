# Go Memory Leak Standards

Reference for the `go-memory-leak-debug`, `perf`, and `code-review` skills. Defines the authoritative catalogue of Go memory leak patterns, severity ratings, detection heuristics, and fix templates.

---

## Severity Scale

| Level | Label | Impact |
|-------|-------|--------|
| P1 | Critical | Unbounded growth — will eventually OOM if not restarted |
| P2 | Significant | Slow but steady growth — service degrades over hours/days |
| P3 | Moderate | Gradual growth bounded under steady load but problematic at scale |

---

## Pattern Catalogue

### P1-1: Goroutine Leak — No Shutdown Path

**Why it leaks:** A goroutine spawned without a mechanism to stop it runs forever. Every spawn creates a stack (minimum 8 KB, grows on demand) that is never reclaimed. At scale, thousands of leaked goroutines consume hundreds of megabytes.

**Detection heuristics:**
```bash
# Goroutines with for-ever loops and no context
grep -n "go func()" --include="*.go" -r . | grep -v "_test.go"
# Then check each site: does the function body select on ctx.Done() or a done channel?

# goroutineCount endpoint check (runtime signal):
curl -s http://localhost:6060/debug/pprof/goroutine?debug=1 | head -30
```

**Anti-pattern:**
```go
// LEAK: goroutine runs forever — no way to stop it
go func() {
    for {
        process(item)
        time.Sleep(interval)
    }
}()
```

**Fix template:**
```go
// Pass a context; select on Done in the loop
go func(ctx context.Context) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            process(item)
        }
    }
}(ctx)
```

---

### P1-2: Goroutine Leak — Blocked Channel

**Why it leaks:** A goroutine blocked permanently on a channel send or receive is never garbage-collected. The goroutine's stack and all values reachable from it are retained.

**Detection heuristics:**
```bash
grep -n "chan " --include="*.go" -r . | grep -v "_test.go"
# Review each: does the reader goroutine exit without draining the channel?
# pprof goroutine profile showing goroutines stuck at "chan receive" or "chan send"
```

**Anti-pattern:**
```go
ch := make(chan Result)
go func() {
    ch <- expensiveOp() // LEAK: if caller exits before receiving, goroutine blocks forever
}()
// caller returns early on timeout — goroutine leaked
```

**Fix template — single send (buffered channel):**

> **Warning:** A buffer of 1 is only safe when the goroutine sends **exactly once**. For multi-send patterns (retry loops, streaming results, etc.) the buffer absorbs the first send and the goroutine blocks on the second — the leak is only delayed, not fixed. Use context cancellation for those cases.

```go
ch := make(chan Result, 1) // safe only for a single-send goroutine
go func() {
    ch <- expensiveOp() // unblocks even if caller is gone — buffer absorbs it
}()
```

**Fix template — multi-send or uncertain send count (context cancellation):**
```go
ch := make(chan Result, 1)
go func(ctx context.Context) {
    for {
        select {
        case <-ctx.Done(): // check before starting expensive work so cancellation is noticed without wasting a full operation cycle
            return
        default:
        }
        result := expensiveOp()
        select {
        case ch <- result:
        case <-ctx.Done():
            return
        }
    }
}(ctx)
```

---

### P1-3: Unbounded Global Map or Slice

**Why it leaks:** A package-level map or slice that accumulates entries without eviction grows for the lifetime of the process. Entries are never GC-eligible as long as the map itself is reachable.

**Detection heuristics:**
```bash
# Package-level map variables
grep -n "^var .*= map\[" --include="*.go" -r . | grep -v "_test.go"
# Package-level slice variables
grep -n "^var .*\[\]" --include="*.go" -r . | grep -v "_test.go"
# Lock + map write without delete
grep -n "\.Lock()\|\.RLock()" --include="*.go" -r . -l
# Check those files for map assignment without eviction logic
```

**Anti-pattern:**
```go
var cache = map[string]*Session{} // LEAK: grows forever, sessions never removed
var mu sync.RWMutex

func store(id string, s *Session) {
    mu.Lock()
    cache[id] = s
    mu.Unlock()
}
```

**Fix template:**
```go
// Option A: TTL-based eviction
type entry struct {
    value   *Session
    expires time.Time
}
var cache = map[string]entry{}

func evict() {
    mu.Lock()
    defer mu.Unlock()
    now := time.Now()
    for k, v := range cache {
        if v.expires.Before(now) {
            delete(cache, k)
        }
    }
}

// Option B: use a bounded LRU (e.g. github.com/hashicorp/golang-lru)
cache, err := lru.New(10000)
if err != nil {
    return fmt.Errorf("failed to create LRU cache: %w", err)
}
```

---

### P1-4: Unclosed HTTP Response Body

**Why it leaks:** `http.Response.Body` is an open network connection. If not closed, the connection is never returned to the transport's idle pool and is eventually leaked. Under load this exhausts file descriptors.

**Detection heuristics:**
```bash
grep -n "http\.Get\|http\.Post\|\.Do(" --include="*.go" -r . | grep -v "_test.go"
# For each hit, check that resp.Body.Close() appears in a defer immediately after error check
grep -n "resp\.Body\.Close\|defer.*Body\.Close" --include="*.go" -r .
```

**Anti-pattern:**
```go
resp, err := http.Get(url)
if err != nil {
    return err
}
// LEAK: resp.Body is never closed if json.Decode fails or function returns early
json.NewDecoder(resp.Body).Decode(&result)
```

**Fix template:**
```go
resp, err := http.Get(url)
if err != nil {
    return err
}
defer func() {
    // Drain before closing so the TCP connection is returned to the pool.
    // Without draining, a partially-read body keeps the connection checked out.
    io.Copy(io.Discard, resp.Body)
    resp.Body.Close()
}()

if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
    return err
}
```

---

### P1-5: Unclosed Database Rows

**Why it leaks:** `*sql.Rows` holds a database connection. If not closed, the connection is never returned to the pool. With `SetMaxOpenConns` set, subsequent queries block; without it, connections accumulate until the DB server rejects them.

**Detection heuristics:**
```bash
grep -n "\.Query\b\|\.QueryContext\b" --include="*.go" -r . | grep -v "_test.go"
# Confirm each hit has a corresponding rows.Close() in a defer
```

**Anti-pattern:**
```go
rows, err := db.QueryContext(ctx, query, args...)
if err != nil {
    return err
}
for rows.Next() { ... }
// LEAK: rows.Close() never called — connection held
```

**Fix template:**
```go
rows, err := db.QueryContext(ctx, query, args...)
if err != nil {
    return err
}
defer rows.Close() // immediately after error check

for rows.Next() {
    if err := rows.Scan(&val); err != nil {
        return err
    }
}
return rows.Err()
```

---

### P1-6: Unclosed gRPC Stream

**Why it leaks:** A gRPC client stream that is never closed keeps the underlying HTTP/2 stream open. The server's goroutine serving the stream also runs indefinitely.

**Detection heuristics:**
```bash
grep -n "\.Send\|\.Recv\|ClientStream\|ServerStream" --include="*.go" -r . | grep -v "_test.go"
# Check for defer stream.CloseSend() on client streams
```

**Anti-pattern:**
```go
stream, err := client.StreamData(ctx)
if err != nil {
    return err
}
// LEAK: stream.CloseSend() never called
for _, item := range items {
    stream.Send(&Request{Data: item})
}
```

**Fix template:**
```go
stream, err := client.StreamData(ctx)
if err != nil {
    return err
}
defer stream.CloseSend()

for _, item := range items {
    if err := stream.Send(&Request{Data: item}); err != nil {
        return err
    }
}
```

---

### P2-1: Timer or Ticker Not Stopped

**Why it leaks:** `time.NewTicker` and `time.NewTimer` allocate a channel and a runtime goroutine to deliver ticks. If `Stop()` is not called, the runtime goroutine runs until the process exits. In functions called frequently this accumulates rapidly.

**Detection heuristics:**
```bash
grep -n "time\.NewTicker\|time\.NewTimer" --include="*.go" -r . | grep -v "_test.go"
# For each, check that Stop() is called (ideally via defer)
```

**Anti-pattern:**
```go
func poll() {
    ticker := time.NewTicker(5 * time.Second) // LEAK: Stop() never called
    for range ticker.C {
        check()
    }
}
```

**Fix template — `time.NewTicker` (repeating):**
```go
func poll(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop() // always stop to release the runtime goroutine

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            check()
        }
    }
}
```

**Fix template — `time.NewTimer` (one-shot):**

`defer timer.Stop()` alone is not sufficient for timers. `Stop()` returns `false` when the timer has already fired before Stop was called — meaning a value is already buffered in the channel. Any subsequent `<-timer.C` will immediately consume that stale value, causing unexpected early firing or a goroutine hang. Drain the channel when Stop reports the timer already fired:

```go
timer := time.NewTimer(5 * time.Second)
defer func() {
    if !timer.Stop() {
        select {
        case <-timer.C: // drain the already-fired value
        default:
        }
    }
}()
```

---

### P2-2: Context Cancel Function Not Called

**Why it leaks:** `context.WithCancel`, `context.WithTimeout`, and `context.WithDeadline` each register the context in an internal tree. If the cancel function is never called, the registration is never removed. Over many requests, this causes steady growth in the runtime's internal context tree.

**Detection heuristics:**
```bash
grep -n "context\.With" --include="*.go" -r . | grep -v "_test.go"
# For each, verify defer cancel() appears immediately after the call
```

**Anti-pattern:**
```go
ctx, cancel := context.WithTimeout(parentCtx, 30*time.Second)
// LEAK: cancel() never called — context tree grows
result, err := doWork(ctx)
```

**Fix template:**
```go
ctx, cancel := context.WithTimeout(parentCtx, 30*time.Second)
defer cancel() // always defer cancel, even when timeout fires naturally

result, err := doWork(ctx)
```

---

### P2-3: sync.Pool Object Retained After Put *(Performance / Race-Safety Anti-pattern)*

> **Classification:** This is primarily a **performance and race-safety anti-pattern**, not a classic memory leak. Memory impact is bounded by the number of concurrent callers — it does not cause the unbounded or steady growth characteristic of P1/P2 leaks.

**Why it matters:** `sync.Pool` exists to reduce allocation pressure by reusing objects across goroutines. If the caller retains a reference to an object after `Put`, two problems arise: (1) the pool cannot reclaim and reuse that object — every subsequent `Get()` allocates a fresh one, defeating the optimization entirely; (2) if another goroutine calls `Get()` and receives the same object while the original caller still holds it, both goroutines share the buffer — a data race with undefined behaviour.

**Detection heuristics:**
```bash
grep -n "\.Put(" --include="*.go" -r . | grep -v "_test.go"
# For each Put call, check two things:
# (a) whether the variable is used after Put in the same function
# (b) whether the variable was passed to a goroutine (go func), stored in a struct field,
#     or passed to a function that may retain it asynchronously before Put was called
```

**Anti-pattern:**
```go
buf := pool.Get().(*bytes.Buffer)
buf.Reset()
doWork(buf)
pool.Put(buf)
// BUG: buf may still be referenced inside doWork's goroutine — race + retained memory
```

**Fix template (safe only when `doWork` is synchronous and does not retain `buf` past its return):**
```go
buf := pool.Get().(*bytes.Buffer)
buf.Reset()
doWork(buf)       // MUST be synchronous; MUST NOT store buf in a field or pass it to a goroutine
buf.Reset()       // clear state before returning to pool
pool.Put(buf)
buf = nil         // nil the local reference to prevent accidental use after Put
```

**When `doWork` is async or may retain `buf` — do not use the pool:**
```go
// Cannot safely pool — ownership of buf transfers to doWork
buf := &bytes.Buffer{}
doWork(buf)       // doWork now owns buf; never Put a buffer whose lifetime you don't control
```

> **Rule:** `buf.Reset()` and `buf = nil` only guard against the *caller's* accidental post-Put access. They cannot prevent a data race if `doWork` holds an internal reference to `buf`. If retention cannot be guaranteed, skip the pool entirely and allocate a fresh buffer.

---

### P2-4: HTTP Client Created Per Request

**Why it leaks:** `&http.Client{}` creates a new `http.Transport` with its own connection pool. When the client falls out of scope, idle connections in that transport are abandoned rather than reused. Under high request rates, this exhausts sockets and prevents connection reuse.

**Detection heuristics:**
```bash
grep -n "http\.Client{" --include="*.go" -r . | grep -v "_test.go"
# Any hit inside a function body (not a package-level var) is a candidate
```

**Anti-pattern:**
```go
func fetch(url string) (*Response, error) {
    client := &http.Client{Timeout: 10 * time.Second} // LEAK: new transport per call
    return client.Get(url)
}
```

**Fix template:**
```go
// package-level or injected shared client
var httpClient = &http.Client{
    Timeout: 10 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     90 * time.Second,
    },
}

func fetch(url string) (*Response, error) {
    return httpClient.Get(url)
}
```

---

### P3-1: String ↔ []byte Conversion in Hot Path

**Why it leaks:** Every `string(b)` or `[]byte(s)` allocates a new backing array. In loops or high-frequency handlers, this generates sustained GC pressure and elevated heap usage even if individual allocations are short-lived.

**Detection heuristics:**
```bash
# Match all common forms: string(b), string([]byte{...}), []byte(s), []byte(stringVar)
grep -nE "string\(\[\]byte|string\([a-zA-Z_][a-zA-Z0-9_]*\)|(\[\]byte)\([a-zA-Z_\"]" --include="*.go" -r . | grep -v "_test.go"
# Flag conversions inside for loops or functions called > 1k/s
```

**Fix template:**
```go
// Use strings.Builder to accumulate without repeated conversion
var b strings.Builder
for _, part := range parts {
    b.WriteString(part)
}
result := b.String()

// Or use unsafe for read-only conversion (measure first):
// func bytesToString(b []byte) string { return *(*string)(unsafe.Pointer(&b)) }
```

---

### P3-2: Slice Append Without Capacity Pre-allocation

Already documented in `standards/go/perf.md`. Elevated here for completeness: repeated doubling in tight loops causes spiky allocation and increases GC pause duration. Under steady load the live heap stabilises, but burst traffic creates transient memory spikes.

**Fix template:**
```go
// Pre-allocate with known or estimated capacity
result := make([]T, 0, len(src))
for _, item := range src {
    result = append(result, transform(item))
}
```

---

### P3-3: Large Struct Passed by Value to Goroutine

**Why it leaks:** Passing a large struct by value to a goroutine copies the struct onto the goroutine's stack (or heap-escapes it). If the goroutine lives long or is spawned frequently, this inflates heap usage without the values ever being reachable for GC.

**Detection heuristics:**
```bash
# Goroutines with struct literal arguments
grep -n "go func(" --include="*.go" -r . | grep -v "_test.go"
# Review: are large structs passed by value?
```

**Fix template:**
```go
// Pass pointer instead of value
go func(cfg *Config) {
    process(cfg)
}(cfg)  // not: }(cfg)  where cfg is Config (value)
```

---

## Detection Quick-Scan Command

Run this as a first pass to find files most likely to contain leaks:

```bash
grep -rn \
  "go func(\|time\.NewTicker\|time\.NewTimer\|context\.With\|http\.Get\|http\.Post\|\.Do(\|db\.Query\|\.QueryContext\|\.Send(\|\.Recv(\|sync\.Pool\|^var .*= map\[" \
  --include="*.go" \
  --exclude-dir=vendor \
  --exclude="*_test.go" \
  -l .
```

Then prioritise files with the highest hit density.

---

## pprof Collection Commands

Use these to get runtime evidence before or after applying fixes:

```bash
# Heap profile (collect two snapshots 5 minutes apart, then diff)
curl -s http://localhost:6060/debug/pprof/heap -o heap1.prof
sleep 300
curl -s http://localhost:6060/debug/pprof/heap -o heap2.prof
go tool pprof -base heap1.prof heap2.prof
# In pprof shell: top20 -cum

# Goroutine count (should be stable under steady load)
curl -s "http://localhost:6060/debug/pprof/goroutine?debug=1" | head -60

# Allocs profile (shows allocation hot spots)
curl -s http://localhost:6060/debug/pprof/allocs -o allocs.prof
go tool pprof allocs.prof
# In pprof shell: top20 -cum

# Mutex contention (if heap is stable but CPU is high)
curl -s http://localhost:6060/debug/pprof/mutex -o mutex.prof
go tool pprof mutex.prof
```

Enable pprof in your service if not already present:

```go
import _ "net/http/pprof"

// In main() or init():
go func() {
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```

---

## Severity Scoring for Ranking

When ranking findings across a codebase, use these base weights:

| Severity | Base weight |
|----------|-------------|
| P1 | 3 |
| P2 | 2 |
| P3 | 1 |

Multiply the base weight by:
- **× 1.5** if the finding is in a component matching the user's suspected area
- **× 1.3** if the finding is in a file modified in recent commits
- **× 2.0** if the finding is confirmed by pprof data (function appears in `top20 -cum`)

> **Exception — P2-3:** Despite its P2 designation, P2-3 (sync.Pool misuse) uses base weight = **1** (P3-equivalent) when scoring. Its impact is bounded by concurrency and is correctness/performance-oriented, not unbounded memory growth.

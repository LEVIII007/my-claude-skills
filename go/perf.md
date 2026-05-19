# Go Performance Standards

## Core Rules

- Avoid unnecessary allocations and copying of large structs.
- Avoid premature optimization — measure before optimizing.
- Critical paths must be benchmarked before and after changes.
- Avoid reflection unless absolutely required — it bypasses type safety and is slow.

## Anti-Patterns to Flag

### Memory Allocations

- Slice appends in a loop without pre-allocated capacity → use `make([]T, 0, len(src))`
- String concatenation in a loop → use `strings.Builder`

### Algorithmic Complexity

- Nested loops over the same collection → likely O(n²): flag it
- Repeated map/slice lookups inside a loop that could be hoisted
- Unbounded recursion without memoisation

### Concurrency

- Sequential HTTP calls that could be parallelised → use `errgroup`

## Benchmark Template

```go
func BenchmarkFunctionName(b *testing.B) {
    input := setupInput()  // realistic, not trivial
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        FunctionName(input)
    }
}
```

Write benchmarks to `<name>_bench_test.go`.

## Benchmark Execution

```bash
go test -bench=BenchmarkFunctionName -benchmem -count=5 ./...
```

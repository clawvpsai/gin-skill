# Concurrency — Goroutines, WaitGroups, Context, Structured Concurrency

## Golden Rules

**Never spawn a goroutine without a way to know when it's done.**
**Treat concurrent work as first-class operations with explicit failure handling.**

## sync.WaitGroup

```go
import "sync"

func processItems(items []string) error {
    var wg sync.WaitGroup
    var errs []error
    
    for _, item := range items {
        wg.Add(1)
        go func(i string) {
            defer wg.Done()
            if err := process(i); err != nil {
                mu.Lock()
                errs = append(errs, err)
                mu.Unlock()
            }
        }(item)
    }
    
    wg.Wait()
    return errors.Join(errs...)
}

var mu sync.Mutex // protects errs
```

## Context Propagation

```go
import "context"

func processWithContext(ctx context.Context, items []string) error {
    for _, item := range items {
        // Check if context is cancelled
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            // Continue processing
        }
        
        if err := processItem(ctx, item); err != nil {
            return err
        }
    }
    return nil
}

// Context with timeout
func handler(c *gin.Context) {
    ctx, cancel := context.WithTimeout(c.Request.Context(), 5*time.Second)
    defer cancel()
    
    result, err := fetchDataWithContext(ctx, "some-url")
    if err != nil {
        if ctx.Err() == context.Canceled {
            c.JSON(499, gin.H{"error": "client cancelled"})
            return
        }
        if ctx.Err() == context.DeadlineExceeded {
            c.JSON(504, gin.H{"error": "operation timed out"})
            return
        }
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(200, result)
}
```

## Worker Pool Pattern

```go
func workerPool(jobs <-chan Job, results chan<- Result, numWorkers int) error {
    var wg sync.WaitGroup
    
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            for job := range jobs {
                result := processJob(job) // blocking work
                results <- result
            }
        }(i)
    }
    
    wg.Wait()
    close(results)
    return nil
}

func main() {
    jobs := make(chan Job, 100)
    results := make(chan Result, 100)
    
    go workerPool(jobs, results, 5) // 5 workers
    
    // Send jobs
    for _, item := range items {
        jobs <- item
    }
    close(jobs)
    
    // Collect results
    for result := range results {
        // process result
    }
}
```

## Channel Patterns

```go
// Fan-out, fan-in
func fanOut(in <-chan int, workers int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for val := range in {
                out <- val * 2
            }
        }()
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}

// Pipeline with cancellation
func pipeline(ctx context.Context, input <-chan int) (<-chan string, <-chan error) {
    out := make(chan string)
    errCh := make(chan error, 1)
    
    go func() {
        defer close(out)
        for n := range input {
            select {
            case <-ctx.Done():
                errCh <- ctx.Err()
                return
            default:
                out <- strconv.Itoa(n * 2)
            }
        }
    }()
    
    return out, errCh
}

// Generator — convert a slice to a channel
func generate(ctx context.Context, items []Item) <-chan Item {
    out := make(chan Item)
    go func() {
        defer close(out)
        for _, item := range items {
            select {
            case <-ctx.Done():
                return
            case out <- item:
            }
        }
    }()
    return out
}
```

## Mutex — Shared State

```go
import "sync"

type Counter struct {
    mu    sync.Mutex
    count int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *Counter) Get() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}

// Or use sync/atomic for simple counters
import "sync/atomic"

var count int64

atomic.AddInt64(&count, 1)
atomic.LoadInt64(&count)

// Atomic for flags
var flag int64

func setFlag() { atomic.StoreInt64(&flag, 1) }
func isSet() bool { return atomic.LoadInt64(&flag) == 1 }
```

## Once — Single Execution

```go
var (
    once     sync.Once
    instance *Service
)

func getInstance() *Service {
    once.Do(func() {
        instance = &Service{}
    })
    return instance
}

// Or sync.OnceValue (Go 1.21+)
var getIngestClient = sync.OnceValue(func() *IngestClient {
    return NewIngestClient()
})
```

## errgroup — Propagating Errors with Context

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) ([]string, error) {
    g, ctx := errgroup.WithContext(ctx)
    
    results := make([]string, len(urls))
    
    for i, url := range urls {
        i, url := i, url // capture range variables
        g.Go(func() error {
            resp, err := http.Get(url)
            if err != nil {
                return err
            }
            defer resp.Body.Close()
            
            body, err := io.ReadAll(resp.Body)
            if err != nil {
                return err
            }
            
            results[i] = string(body)
            return nil
        })
    }
    
    if err := g.Wait(); err != nil {
        return nil, err
    }
    
    return results, nil
}

// errgroup in Gin handler — parallel DB queries with shared context
func getUserDashboard(c *gin.Context) {
    userID := c.GetString("user_id")
    
    g, ctx := errgroup.WithContext(c.Request.Context())
    var user *User
    var posts []Post
    var notifications []Notification
    
    g.Go(func() error {
        var err error
        user, err = userRepo.GetByID(ctx, userID)
        return err
    })
    g.Go(func() error {
        var err error
        posts, err = postRepo.GetByUserID(ctx, userID, 10)
        return err
    })
    g.Go(func() error {
        var err error
        notifications, err = notificationRepo.GetUnread(ctx, userID)
        return err
    })
    
    if err := g.Wait(); err != nil {
        c.JSON(500, gin.H{"error": "failed to load dashboard"})
        return
    }
    
    c.JSON(200, gin.H{
        "user":          user,
        "posts":         posts,
        "notifications": notifications,
    })
}
```

## Structured Concurrency Patterns

### Timeout per Task in Batch

```go
func batchWithTimeout(ctx context.Context, items []Item, perItemTimeout time.Duration) ([]Result, error) {
    results := make([]Result, 0, len(items))
    
    for _, item := range items {
        itemCtx, cancel := context.WithTimeout(ctx, perItemTimeout)
        
        result, err := processItem(itemCtx, item)
        cancel() // always cancel to prevent context leak
        
        if err != nil {
            log.Printf("item %v failed: %v", item.ID, err)
            continue // don't fail entire batch for one item
        }
        results = append(results, result)
    }
    
    return results, nil
}
```

### Semaphore — Limit Concurrent Operations

```go
import "golang.org/x/sync/semaphore"

func limitedFetch(ctx context.Context, items []Item, maxConcurrent int) error {
    sem := semaphore.NewWeighted(int64(maxConcurrent))
    
    var wg sync.WaitGroup
    for _, item := range items {
        item := item // capture
        wg.Add(1)
        
        // Acquire before spawning goroutine (limits concurrency)
        if err := sem.Acquire(ctx, 1); err != nil {
            wg.Done()
            return ctx.Err()
        }
        
        go func() {
            defer wg.Done()
            defer sem.Release(1)
            
            if err := processItem(ctx, item); err != nil {
                log.Printf("item %v failed: %v", item.ID, err)
            }
        }()
    }
    
    wg.Wait()
    return nil
}
```

### Context-Sensitive Fan-Out

```go
// Process items in parallel, but stop on first error
func fanOutWithFailFast(ctx context.Context, items []Item, workers int) error {
    ctx, cancel := context.WithCancelCause(ctx)
    defer cancel()
    
    g, ctx := errgroup.WithContext(ctx)
    
    itemCh := make(chan Item, len(items))
    for _, item := range items {
        itemCh <- item
    }
    close(itemCh)
    
    for range workers {
        g.Go(func() error {
            for item := range itemCh {
                select {
                case <-ctx.Done():
                    return ctx.Err()
                default:
                }
                
                if err := processItem(ctx, item); err != nil {
                    cancel(err) // cancel all other workers
                    return err
                }
            }
            return nil
        })
    }
    
    return g.Wait()
}
```

## slices and maps (Go 1.21+) — Concurrent-Friendly Patterns

Go 1.21+ added `maps` and `slices` packages with thread-safe copy utilities:

```go
import (
    "maps"
    "slices"
)

// Clone slices safely — better than manual copy
func getActiveUsers(users []User) []*User {
    active := slices.Filter(users, func(u User) bool {
        return u.Active
    })
    // slices are already value types — no race if read-only
    return slices.Clone(active)
}

// Clone maps for safe read-only copies
func getPermissionsMap(roles map[string][]string) map[string][]string {
    return maps.Clone(roles)
}

// Search/sort patterns
func findUser(users []User, id uint) *User {
    idx := slices.IndexFunc(users, func(u User) bool { return u.ID == id })
    if idx < 0 {
        return nil
    }
    return &users[idx]
}

// Batch process with worker pool using slices
func processBatches(items []Item, workers int) []Result {
    jobs := make(chan Item, len(items))
    results := make(chan Result, len(items))
    
    for _, item := range items {
        jobs <- item
    }
    close(jobs)
    
    var wg sync.WaitGroup
    for range workers {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }()
    }
    
    wg.Wait()
    close(results)
    
    // Collect results — slices.Concat for joining result batches
    var allResults []Result
    for r := range results {
        allResults = append(allResults, r)
    }
    
    return allResults
}
```

## Goroutine Leak Profiler (Go 1.26+)

Go 1.26 introduces an experimental goroutine leak profiler that detects leaked goroutines in production.

```bash
# Enable at build time
GOEXPERIMENT=goroutineleakprofile go build -o myapp .

# Run with the build, then access via pprof
./myapp
# curl http://localhost:6060/debug/pprof/goroutine?debug=1
```

**Via Gin with `net/http/pprof`:**
```go
import _ "net/http/pprof"

func main() {
    r := gin.Default()
    go func() {
        // pprof server on separate port
        http.ListenAndServe(":6060", nil)
    }()
    r.Run()
}
```

**Common leak patterns the profiler catches:**
- Goroutines blocked on `channel send/receive` with no sender/receiver
- Goroutines blocked on `sync.Mutex` or `sync.RWMutex`  
- Goroutines in `time.Sleep` with no wakeup mechanism
- Goroutines waiting on `syscall` with no response

**Tip:** Combine with `pprof.Lookup("goroutine")` in production to dump live goroutine stacks on demand.

**Leak detection in tests:**
```go
// Detect goroutine leaks in tests
func TestNoGoroutineLeak(t *testing.T) {
    before := runtime.NumGoroutine()
    
    // Run your concurrent code
    doWork()
    
    // Wait for goroutines to settle
    time.Sleep(100 * time.Millisecond)
    
    after := runtime.NumGoroutine()
    if after > before {
        t.Errorf("goroutine leak: before=%d, after=%d", before, after)
    }
}
```

## Common Mistakes

1. **Goroutine without WaitGroup or context** — memory leak, no way to know when done
2. **Not checking context cancellation** — long-running goroutines waste CPU
3. **Mutex contention** — too many locks in hot paths degrade performance
4. **Closing channels twice** — panic, only close in sender (or use `defer close()` in sender goroutine)
5. **Sending to nil channel** — blocks forever, causes deadlock
6. **Not using `errgroup`** — errors from goroutines silently lost
7. **Cloning slices with pre-allocated copy manually** — use `slices.Clone()` instead (Go 1.21+)
8. **Manually copying maps in loops** — use `maps.Clone()` instead (Go 1.21+)
9. **Not calling `cancel()` on derived contexts** — context leak, prevent GC of context resources
10. **Using mutex for everything** — consider atomic operations for simple counters/flags
11. **Spawning unlimited goroutines for unbounded work** — use semaphore or worker pool
12. **Not handling errors in errgroup** — first error cancels context, but you must check `g.Wait()`

---

## Updated from Research (2026-05)

### Go 1.21+ slices/maps Packages
- `slices.Clone()` and `maps.Clone()` are the idiomatic way to copy collections
- `slices.Filter`, `slices.IndexFunc`, `slices.Concat` reduce boilerplate in concurrent data processing
- `sync.OnceValue` (Go 1.21+) replaces manual `sync.Once` + initialization patterns

### Structured Concurrency
- Semaphore via `golang.org/x/sync/semaphore` limits concurrent goroutines for bounded resource usage
- `context.WithCancelCause` (Go 1.21+) allows attaching error to cancellation — useful for fan-out patterns
- Use `errors.Join` to collect multiple errors from parallel operations

### Go 1.26 Goroutine Leak Profiler
- Enable with `GOEXPERIMENT=goroutineleakprofile` at build time
- Pairs well with `net/http/pprof` for on-demand goroutine stack dumps in production
- Add `time.Sleep` in tests to allow goroutines to settle before counting

### errgroup Integration with Gin
- errgroup's context cancellation integrates with Gin handler context — first failure cancels remaining parallel operations
- Use `errgroup.WithContext` at the start of parallel DB/API calls to propagate errors cleanly

### Sources
- https://pkg.go.dev/slices
- https://pkg.go.dev/maps
- https://go.dev/doc/go1.26
- https://pkg.go.dev/golang.org/x/sync/errgroup
- https://pkg.go.dev/golang.org/x/sync/semaphore
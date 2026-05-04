# Concurrency — Goroutines, WaitGroups, Context

## Golden Rules

**Never spawn a goroutine without a way to know when it's done.**

## sync.WaitGroup

```go
import "sync"

func processItems(items []string) {
    var wg sync.WaitGroup
    
    for _, item := range items {
        wg.Add(1)
        go func(i string) {
            defer wg.Done()
            process(i)
        }(item)
    }
    
    wg.Wait() // blocks until all goroutines finish
}
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
func workerPool(jobs <-chan Job, results chan<- Result, numWorkers int) {
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

// Pipeline
func pipeline(input <-chan int) <-chan string {
    out := make(chan string)
    go func() {
        defer close(out)
        for n := range input {
            out <- strconv.Itoa(n * 2)
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
```

## errgroup — Propagating Errors

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

# Or set at runtime (requires the build flag)
# Then access via pprof: /debug/pprof/goroutine?debug=1
```

**Via Gin with `net/http/pprof`:**
```go
import _ "net/http/pprof"

func main() {
    r := gin.Default()
    go func() {
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

## Common Mistakes

1. **Goroutine without WaitGroup** — memory leak, no way to know when done
2. **Not checking context cancellation** — long-running goroutines waste CPU
3. **Mutex contention** — too many locks in hot paths degrade performance
4. **Closing channels twice** — panic, only close in sender (or use `defer close()` in sender goroutine)
5. **Sending to nil channel** — blocks forever, causes deadlock
6. **Not using `errgroup`** — errors from goroutines silently lost
7. **Cloning slices with pre-allocated copy manually** — use `slices.Clone()` instead (Go 1.21+)
8. **Manually copying maps in loops** — use `maps.Clone()` instead (Go 1.21+)

---

## Updated from Research (2026-05)

### Go 1.21+ slices/maps Packages
- `slices.Clone()` and `maps.Clone()` are the idiomatic way to copy collections (safer, cleaner than manual loops)
- `slices.Filter`, `slices.IndexFunc`, `slices.Concat` reduce boilerplate in concurrent data processing
- errgroup with Gin handlers is a clean pattern for parallel I/O (DB queries, external API calls)

### Go 1.26 Goroutine Leak Profiler
- Enable with `GOEXPERIMENT=goroutineleakprofile` at build time
- Pairs well with `net/http/pprof` for on-demand goroutine stack dumps in production

### Sources
- https://pkg.go.dev/slices
- https://pkg.go.dev/maps
- https://go.dev/doc/go1.26
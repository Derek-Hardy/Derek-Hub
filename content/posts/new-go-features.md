+++
date = '2026-05-26T12:30:00+08:00'
draft = false
title = 'New Go Features 1.18 - 1.26'
+++

I started writing Go at GS with version `1.17`, so `1.18` is roughly where "modern Go" begins for me.

Since then, Go has added quite a few features that make day-to-day engineering work more pleasant. 
I will focus on the parts that affect normal backend/service code: concurrency, error handling, testing, collections, routing and tools.

### sync.WaitGroup.Go (1.25)

Before Go 1.25, the common `WaitGroup` pattern looks like this:

```go
var wg sync.WaitGroup

for _, job := range jobs {
	job := job

	wg.Add(1)
	go func() {
		defer wg.Done()
		process(job)
	}()
}

wg.Wait()
```

This is not terrible, but it is easy to put `Add` in the wrong place or forget `Done`.
The same boilerplate also appears in many places once a codebase has many goroutines.

Go 1.25 added `WaitGroup.Go`:

```go
var wg sync.WaitGroup

for _, job := range jobs {
	job := job

	wg.Go(func() {
		process(job)
	})
}

wg.Wait()
```

Conceptually, `wg.Go(fn)` means:

```go
wg.Add(1)
go func() {
	defer wg.Done()
	fn()
}()
```

It is a small feature, but it puts the usual structure directly into the API.

One thing to remember: `WaitGroup` only waits. It does not collect errors or cancel sibling goroutines. If I need error propagation, I still prefer `errgroup`:

```go
g, ctx := errgroup.WithContext(ctx)

for _, job := range jobs {
	job := job

	g.Go(func() error {
		return process(ctx, job)
	})
}

if err := g.Wait(); err != nil {
	return err
}
```

### tool directive (1.24)

This is a handy tooling improvement.

Many Go projects depend on executable tools: linters, protobuf generators and so on. Before Go 1.24, a common workaround was to create a `tools.go` file:

```go
//go:build tools

package tools

import (
	_ "google.golang.org/protobuf/cmd/protoc-gen-go"
	_ "google.golang.org/grpc/cmd/protoc-gen-go-grpc"
)
```

The blank import keeps the tool in `go.mod`, but it is a little indirect. New joiners often need to ask why this file exists.

Go 1.24 lets `go.mod` track executable dependencies directly with `tool` directives:

```bash
go get -tool google.golang.org/protobuf/cmd/protoc-gen-go
go get -tool google.golang.org/grpc/cmd/protoc-gen-go-grpc
```

Then `go.mod` can contain something like:

```go
go 1.24

tool (
	google.golang.org/protobuf/cmd/protoc-gen-go
	google.golang.org/grpc/cmd/protoc-gen-go-grpc
)
```

Now we can check the plugin versions through the Go toolchain:

```bash
go tool protoc-gen-go --version
go tool protoc-gen-go-grpc --version
```

This is cleaner because executable dependencies are now first-class module metadata.

### range over integers and better http.ServeMux (1.22)

Go 1.22 made two small changes that are easy to appreciate in everyday code.

First, `for range` can now range over integers:

```go
for i := range 5 {
	fmt.Println(i)
}

// 0
// 1
// 2
// 3
// 4
```

Before that, we would write the usual C-style loop:

```go
for i := 0; i < 5; i++ {
	fmt.Println(i)
}
```

The old form is still fine, especially when I need custom steps. But `range 5` is concise for "repeat this N times" or "loop from 0 to N-1".

For example:

```go
func retry(times int, fn func() error) error {
	var err error

	for range times {
		if err = fn(); err == nil {
			return nil
		}
	}

	return err
}
```

The same release also made `http.ServeMux` more useful.

Previously, the standard library mux was quite basic, so many projects reached for `chi`, `gorilla/mux`, `gin`, or other routers. Go 1.22 added method matching and path wildcards:

```go
mux := http.NewServeMux()

mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")
	fmt.Fprintf(w, "get user %s", id)
})

mux.HandleFunc("POST /users", func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "create user")
})

http.ListenAndServe(":8080", mux)
```

This makes the standard library good enough for many smaller services.

I still tend to use a dedicated router when I need middleware groups. But for a small internal API, the standard `ServeMux` is now much more practical.

### slices, min, max and clear (1.21)

Go 1.21 is probably the release I notice most often in normal application code.

The new `slices` package collects many operations that people used to implement themselves.
It helps to remove a lot of tiny helper functions from application code.

```go
names := []string{"bob", "alice", "chris"}

slices.Sort(names)
fmt.Println(names) // [alice bob chris]

fmt.Println(slices.Contains(names, "alice")) // true
fmt.Println(slices.Index(names, "chris"))    // 2
```

It also works well with structs:

```go
type User struct {
	Name  string
	Score int
}

users := []User{
	{Name: "alice", Score: 80},
	{Name: "bob", Score: 95},
	{Name: "chris", Score: 70},
}

slices.SortFunc(users, func(a, b User) int {
	return cmp.Compare(b.Score, a.Score) // descending
})

fmt.Println(users[0].Name) // bob
```

This is a good example of generics improving the standard library. Before generics, `slices.SortFunc` could not exist in this form without losing type safety or relying on reflection.

Go 1.21 also added `min`, `max`, and `clear` as builtins.

```go
lowest := min(10, 3, 8)
highest := max(10, 3, 8)

fmt.Println(lowest)  // 3
fmt.Println(highest) // 10
```

`clear` works on maps and slices:

```go
counts := map[string]int{
	"success": 10,
	"failure": 2,
}

clear(counts)
fmt.Println(counts) // map[]
```

For slices, `clear` sets elements to their zero value but keeps the length:

```go
data := []int{1, 2, 3}

clear(data)
fmt.Println(data) // [0 0 0]
```

### errors.Join (1.20)

Before Go 1.20, error wrapping was mostly one error wrapping another error:

```go
if err != nil {
	return fmt.Errorf("load config: %w", err)
}
```

Sometimes multiple things can fail, and all of them are worth reporting. Cleanup is a classic example:

```go
func closeAll(db *sql.DB, cache io.Closer, logs io.Closer) error {
	return errors.Join(
		db.Close(),
		cache.Close(),
		logs.Close(),
	)
}
```

`errors.Join` returns one error that wraps multiple errors. `nil` errors are ignored, so this is convenient for cleanup paths.

It also works with `errors.Is` and `errors.As`:

```go
err := errors.Join(
	fmt.Errorf("write audit log: %w", os.ErrPermission),
	fmt.Errorf("close file: %w", fs.ErrClosed),
)

fmt.Println(errors.Is(err, os.ErrPermission)) // true
fmt.Println(errors.Is(err, fs.ErrClosed))     // true
```

The conceptual change: an error can now represent a tree, not just a chain.

### Atomic types (1.19)

Before Go 1.19, `sync/atomic` was mostly a set of functions:

```go
var count int64

atomic.AddInt64(&count, 1)
fmt.Println(atomic.LoadInt64(&count))
```

This works, but the raw variable is still visible. Someone can accidentally read or write it without using atomic operations.

Go 1.19 added typed atomic values:

```go
var count atomic.Int64

count.Add(1)
fmt.Println(count.Load())
```

Now the value is hidden behind methods, so the correct API is the natural one to use.

A common use case is a counter:

```go
type Metrics struct {
	requests atomic.Int64
	failures atomic.Int64
}

func (m *Metrics) Record(err error) {
	m.requests.Add(1)

	if err != nil {
		m.failures.Add(1)
	}
}

func (m *Metrics) Snapshot() (requests int64, failures int64) {
	return m.requests.Load(), m.failures.Load()
}
```

### Fuzzing in go test (1.18)

Fuzzing is a way to test code with generated inputs.

Normal unit tests check the examples I can think of. Fuzzing checks examples I probably would not think of.

A fuzz test starts with seed inputs, then the fuzzing engine mutates them to find crashes, panics, or invalid assumptions.

For example, suppose I have a parser:

```go
func ParseUserID(s string) (int64, error) {
	if s == "" {
		return 0, errors.New("empty user id")
	}

	return strconv.ParseInt(s, 10, 64)
}
```

I can fuzz it like this:

```go
func FuzzParseUserID(f *testing.F) {
	f.Add("1")
	f.Add("12345")
	f.Add("-1")
	f.Add("")

	f.Fuzz(func(t *testing.T, input string) {
		id, err := ParseUserID(input)
		if err != nil {
			return
		}

		if strconv.FormatInt(id, 10) != input {
			t.Fatalf("round trip failed: input=%q id=%d", input, id)
		}
	})
}
```

Run the seed corpus like a normal test:

```bash
go test ./...
```

Run fuzzing for a period of time:

```bash
go test -fuzz=FuzzParseUserID -fuzztime=30s
```
For input-heavy code, it is a useful tool to have inside the standard `go test` workflow.

### Generics and library adoption (1.18+)

Generics are the biggest language change in this period.

Before Go 1.18, reusable code often had to choose between:

- writing the same logic for multiple concrete types
- using `interface{}` and type assertions
- using reflection

Generics introduced type parameters, so we can write reusable code while keeping type safety.

#### Functional style

The most obvious use case is collection-style helpers.
To keep the idea simple, here is a function that returns the first item matching a condition:

```go
func First[T any](items []T, ok func(T) bool) (T, bool) {
	for _, item := range items {
		if ok(item) {
			return item, true
		}
	}

	var zero T
	return zero, false
}
```

There is only one type parameter here.
If the input is `[]int`, `T` is `int`. If the input is `[]string`, `T` is `string`.

```go
numbers := []int{3, 8, 2, 10}

n, found := First(numbers, func(n int) bool {
	return n > 5
})

fmt.Println(n, found) // 8 true
```

The same function also works for strings:

```go
names := []string{"alice", "bob", "chris"}

name, found := First(names, func(name string) bool {
	return strings.HasPrefix(name, "c")
})

fmt.Println(name, found) // chris true
```

This looks natural if you come from Python or Java streams. In Go, I still try not to overdo it. A plain `for` loop is often clearer.

But for small reusable search/filter/transform helpers, generics are quite handy.

#### OOP-ish design

Go is not a classical OOP language, but it has structs, methods and interfaces. Generics can make reusable typed containers cleaner.

For example, a typed cache:

```go
type Cache[K comparable, V any] struct {
	mu   sync.RWMutex
	data map[K]V
}

func NewCache[K comparable, V any]() *Cache[K, V] {
	return &Cache[K, V]{
		data: make(map[K]V),
	}
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()

	v, ok := c.data[key]
	return v, ok
}

func (c *Cache[K, V]) Set(key K, value V) {
	c.mu.Lock()
	defer c.mu.Unlock()

	c.data[key] = value
}
```

Now the cache is reusable but still typed:

```go
users := NewCache[string, User]()

users.Set("u-123", User{Name: "alice"})

user, ok := users.Get("u-123")
if ok {
	fmt.Println(user.Name)
}
```

#### Better standard library adoption

The useful part is that generics did not stay only as a language feature. Later standard library packages adopted it.

Before `slices`, I might write a small helper for every concrete type:

```go
func ContainsInt64(items []int64, target int64) bool {
	for _, item := range items {
		if item == target {
			return true
		}
	}

	return false
}

func ContainsString(items []string, target string) bool {
	for _, item := range items {
		if item == target {
			return true
		}
	}

	return false
}
```

The logic is exactly the same, but the types are different. This kind of helper is small, but it quietly spreads across a codebase.

With `slices.Contains`, the standard library can express that shared logic once:

```go
ids := []int64{3, 1, 2}
names := []string{"alice", "bob", "chris"}

fmt.Println(slices.Contains(ids, int64(2)))      // true
fmt.Println(slices.Contains(names, "derek"))     // false
fmt.Println(slices.Index(names, "chris"))        // 2
```

It also works for ordering:

```go
ids := []int64{3, 1, 2}
slices.Sort(ids)
fmt.Println(ids) // [1 2 3]
```

`slices`, `maps` and `cmp` are good examples of generic library design. They made common helpers type-safe and reusable without each project inventing its own utility package.

My current takeaway is:

- use generics for containers, algorithms and type-safe reusable helpers
- avoid forcing generics into business logic where a normal interface or simple function is clearer
- prefer standard library generic helpers before writing my own

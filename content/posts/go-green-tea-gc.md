+++
date = '2026-05-27T12:45:00+08:00'
draft = true
title = 'Go 1.26 Green Tea GC'
+++

When I was writing the summary for new Go features from `1.18` to `1.26`, I initially put Green Tea GC into the list.

But it feels different from features like `slices`, `errors.Join`, `tool` directives or `WaitGroup.Go`. Those are APIs I can directly call. Green Tea is a runtime change. Ideally, most Go programs just get the benefit after upgrading to Go `1.26`.

Still, I want to understand it a bit better because GC behavior matters a lot for backend services. It affects latency, CPU usage, memory pressure and the mysterious production question: "Why did this service suddenly spend more time in runtime?"

### What problem does GC solve?

Go manages heap memory automatically.

When a value is allocated on the heap, the program can keep using it as long as it is reachable. Once it is no longer reachable, the runtime can reclaim that memory later.

The simplified GC flow is:

1. Start from roots such as goroutine stacks and global variables.
2. Follow pointers from those roots to find reachable heap objects.
3. Mark reachable objects as live.
4. Sweep memory that was not marked as live.

This is usually called mark-sweep garbage collection.

The nice part is that application code does not need to manually free memory. The cost is that the runtime has to periodically do extra work to find what is still alive.

### Why is the marking phase expensive?

The marking phase is basically graph traversal.

Imagine this data structure:

```go
type Node struct {
	ID       int
	Parent   *Node
	Children []*Node
	Metadata map[string]string
}
```

If the application keeps a pointer to one `Node`, the GC has to scan that object and then follow pointers to its parent, children, slice backing array, map buckets, strings and so on.

The hard part is that these objects may not sit next to each other in memory.

Conceptually it may look like this:

```text
stack root
   |
   v
object A  --->  object B  --->  object C
  page 1        page 8         page 3
```

The old mental model is: find an object, scan it, follow its pointers, scan the next object, and keep going.

This works logically, but it can be unfriendly to modern CPUs. CPUs are fast when they can read nearby memory repeatedly. They are much slower when the program keeps jumping to unrelated memory locations.

So the GC may spend a lot of time waiting on memory, even though the CPU itself is powerful.

### Locality matters

CPU caches are one of those details I know about, but do not always think about when writing normal Go code.

When the CPU reads memory, it tends to bring nearby memory into cache as well. If the next operation also needs nearby memory, it is fast. If the next operation jumps somewhere far away, it may need to wait for main memory again.

For normal application code, we sometimes optimize around this without naming it:

```go
type Event struct {
	UserID int64
	Score  int64
}

events := []Event{
	{UserID: 1, Score: 10},
	{UserID: 2, Score: 20},
	{UserID: 3, Score: 30},
}

var total int64
for _, event := range events {
	total += event.Score
}
```

This is cache-friendly because the slice stores elements contiguously.

Compare that with chasing pointers:

```go
type Event struct {
	UserID int64
	Score  int64
	Next   *Event
}

for event := head; event != nil; event = event.Next {
	total += event.Score
}
```

This can be perfectly fine, but the objects may be scattered around the heap. The CPU has less ability to predict and cache the next access.

GC marking has a similar challenge. It often has to follow pointers through a large object graph.

### What does Green Tea change?

The short version: Green Tea tries to work more with pages instead of individual objects.

In the old object-oriented scanning model, the GC work list is mostly about objects to scan. That can lead to many small pieces of work across different parts of the heap.

Green Tea changes the unit of scanning work toward memory pages. Instead of immediately jumping from object to object across the heap, it tries to collect and scan work by page.

The official blog summarizes the core idea as "work with pages, not objects".

My own mental model is:

```text
Before:
scan object A on page 1
scan object B on page 8
scan object C on page 3
scan object D on page 8

Green Tea:
notice there is work on page 1, page 3, page 8
scan more objects on the same page together
reduce random jumping
```

This should improve:

- cache locality
- memory bandwidth usage
- CPU scalability when multiple GC workers are marking objects
- overhead from managing many tiny units of GC work

The goal is not to change what is considered live. The goal is to do the same marking work in a way that better matches modern hardware.

### Why small objects matter

The Go 1.26 release notes specifically mention marking and scanning small objects.

That makes sense to me because backend services often allocate many small objects:

```go
type RequestContext struct {
	RequestID string
	UserID    int64
	Headers   map[string]string
	StartedAt time.Time
}
```

Or many small structs during request processing:

```go
type ValidationError struct {
	Field   string
	Message string
}

func validate(input Input) []ValidationError {
	var errs []ValidationError

	if input.Email == "" {
		errs = append(errs, ValidationError{
			Field:   "email",
			Message: "email is required",
		})
	}

	return errs
}
```

None of these allocations are suspicious by themselves. But at high QPS, small heap objects can create a lot of GC work.

Green Tea is designed to make that marking/scanning work cheaper, especially when there are many small objects spread across the heap.

### What changes in my code?

Usually, nothing.

In Go `1.26`, Green Tea is enabled by default:

```bash
go test ./...
go build ./...
```

If there is a suspected regression, it can be disabled temporarily at build time:

```bash
GOEXPERIMENT=nogreenteagc go test ./...
```

This opt-out exists so teams can compare behavior. I would not use it as a normal setting unless I had a concrete benchmark or production signal showing a problem.

### How I would evaluate it in a service

I would not expect every service to suddenly become dramatically faster.

If a service spends very little time in GC, then reducing GC overhead will not move the total CPU much.

For example:

```text
Service A:
total CPU time = 100
GC CPU time    = 5

Even a 20% GC improvement saves around 1 total CPU unit.
```

But if a service allocates heavily:

```text
Service B:
total CPU time = 100
GC CPU time    = 25

A 20% GC improvement saves around 5 total CPU units.
```

That can be meaningful.

In practice, I would look at:

- CPU profiles before and after upgrade
- time spent in `runtime.gcBgMarkWorker`
- allocation rate
- p95 and p99 latency
- heap size and GC frequency
- service-level CPU usage under similar traffic

For local experiments, I can still use the usual profiling tools:

```bash
go test -bench=. -benchmem ./...
go test -run=^$ -bench=BenchmarkHotPath -cpuprofile=cpu.out ./...
go tool pprof cpu.out
```

### What Green Tea does not solve

Green Tea makes GC work cheaper, but it does not make allocations free.

I still want to avoid unnecessary heap allocations in hot paths:

```go
// Less ideal in a hot path if this escapes repeatedly.
func labels(userID int64) map[string]string {
	return map[string]string{
		"user_id": strconv.FormatInt(userID, 10),
	}
}
```

Sometimes a simple struct or caller-provided buffer is better:

```go
type Labels struct {
	UserID string
}

func labels(userID int64) Labels {
	return Labels{
		UserID: strconv.FormatInt(userID, 10),
	}
}
```

Of course, I would only do this when profiling says it matters. Most code should remain simple and readable.

The important point: Green Tea reduces GC overhead. It does not remove the general engineering habit of checking allocations in performance-sensitive code.

### My takeaway

The most interesting part of Green Tea is that it is not a new GC algorithm from the application developer's point of view. It still finds reachable objects and reclaims unreachable memory.

The change is more about hardware sympathy.

Instead of treating marking as a pure object graph problem, Green Tea treats memory layout as part of the problem. By scanning more work by page, it gets better locality and better CPU scaling.

For me, the practical summary is:

- Go `1.26` enables Green Tea by default
- most application code does not need to change
- allocation-heavy services are more likely to benefit
- profiling is still the right way to judge the impact
- reducing unnecessary allocations still matters

Useful links:

- [Go 1.26 release notes](https://go.dev/doc/go1.26)
- [The Green Tea Garbage Collector](https://go.dev/blog/greenteagc)

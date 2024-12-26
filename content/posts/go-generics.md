+++
date = '2024-02-16T21:39:17+08:00'
draft = false
title = 'A peek into Go Generics'
+++

Since version `1.18`, Go has finally introduced the support for generics, together with `any` as a type alias of `interface{}`

It gives more flexibility for user to create a general collection for similar data structures. Previously we have no choice but
to write up the same `struct` for each data type we want to support.

The syntax can look a bit strange if you are already used to the old-school Go code. 
Or maybe it's just because I haven't used it often enough.

### Generic struct

For example, I want to create a `Cache` data structure that allows me to look up for compute resources available to execute workloads.
However, I have two types of such resources, so I want my `Cache` and its methods to accept both data types.

```go
type HostStatus int

// define enum values for the type HostStatus
const (
	statusFree HostStatus = iota
	statusOccupied
	statusDecommissioned
)

// I've got Host that represents a single physical host
type Host struct {
	Name   string
	Status HostStatus // it can be free, occupied or already shut down
}

func (h Host) GetName() string {
    return h.Name
}

// I've also got resource pool that represents a cluster of multiple hosts
type Pool struct {
	Name     string
	IsActive bool // it may or may not be active to accept workloads
}

func (p Pool) GetName() string {
    return p.Name
}
```

I can now create one generic `Cache` to contain either of the above types.

```go
// define the resource interface here as a set of types, 
// plus GetName method has to be implemented 
type Resource interface {
	Host | Pool
	GetName() string
}

// define the generic Cache with type parameter of Resource
type Cache[T Resource] struct {
	data map[string]*T
	mu   sync.Mutex
}

func (q *Cache[T]) Lookup(name string) *T {
    return q.data[name]
}

func (q *Cache[T]) Update(resources []T) {
    q.mu.Lock()
    defer q.mu.Unlock()

    for _, r := range resources {
        q.data[r.GetName()] = &r
    }
}
```

Now let's use it in action:

```go
func main() {
	c := Cache[Host]{
		data: make(map[string]*Host),
	}

	hosts := []Host{
		{
			Name:   "host-1",
			Status: statusFree,
		},
		{
			Name:   "host-2",
			Status: statusOccupied,
		},
	}

	c.Update(hosts)

	fmt.Println(c.Lookup("host-1")) // &{host-1 0}
	fmt.Println(c.Lookup("host-2")) // &{host-2 1}
    fmt.Println(c.Lookup("host-3")) // nil
}
```

The same `Cache` can be used for `Pool` as well. Let's use it again:

```go
func main() {
	pc := Cache[Pool]{
		data: make(map[string]*Pool),
	}

	pools := []Pool{
		{
			Name:     "pool-1",
			IsActive: true,
		},
		{
			Name:     "pool-2",
			IsActive: false,
		},
	}

	pc.Update(pools)

	fmt.Println(pc.Lookup("pool-1")) // &{pool-1 true}
	fmt.Println(pc.Lookup("pool-2")) // &{pool-2 false}
	fmt.Println(pc.Lookup("pool-3")) // nil
}
```

### Generic function

Since the introduction of generics, new packages in Go added in later versions also start to make use of the concept.

Take `package slices` for example:

```go
// Index returns the index of the first occurrence of v in s,
// or -1 if not present.
func Index[S ~[]E, E comparable](s S, v E) int {
	for i := range s {
		if v == s[i] {
			return i
		}
	}
	return -1
}
```

The generic expression `S ~[]E, E comparable` means that `S` can be any type whose underlying type is a slice of `E`, including `[]E` itself.
And element `E` has to implement `comparable` interface, which is a builtin interface implemented by basic types such as `numbers`, `strings` (so that equality operation can be performed).

Many functions in the `slices` package have a more general counterpart. Take `IndexFunc` for example, the element `E` can be anything, but it is
now your responsibility to define the custom matching criteria in the form of a function parameter.

```go
// IndexFunc returns the first index i satisfying f(s[i]),
// or -1 if none do.
func IndexFunc[S ~[]E, E any](s S, f func(E) bool) int {
	for i := range s {
		if f(s[i]) {
			return i
		}
	}
	return -1
}
```

Overall, I don't think Generics is used as often in my Go projects as the case in Java. 
However, I do find it come in handy when I occasionally have multiple entities that need to be managed in a similar way.
+++
date = '2023-12-17T15:02:58+08:00'
draft = false
title = 'Data Structure in Go - Part 2'
+++

It is true that Go is designed with a minimalistic touch. It does not have a rich set of collections or in-built data structures like Java, 
where you can just import `java.util.PriorityQueue` or `java.util.Stack` and start using them.

However, besides the basic `map` and `slice`, Go does come with a builtin `container` package that we should at least be aware  and 
make use of it to implement things like `Stack`, `Queue`, `PriorityQueue` and `Deque`.

### container/list

Go has a doubly linked list implementation. Each node looks like:

```go
type Element struct {
	prev, next *Element
	Value any
}
```

Most fundamentally, it supports to append and peek the `Element` from the tail of the list, which means we can implement a Stack:

```go
import "container/list"

type Stack struct {
	data *list.List
}

func NewStack() *Stack {
	return &Stack{
		data: list.New(),
    }
}

func (s *Stack) Push(value any) {
	s.data.PushBack(value) // push element in from the back i.e. append
}

func (s *Stack) Pop() any {
	elem := s.data.Back()
	return s.data.Remove(elem)
}

func (s *Stack) Top() any {
	elem := s.data.Back() // stack is first in, last out
    return elem.Value
}
```

The `List` also allows us to peek from the head. We can also build a queue from it:

```go
type Queue struct {
	data *list.List
}

func NewQueue() *Queue {
	return &Queue{
		data: list.New(),
    }
}

// let's build a queue of integers
func (q *Queue) Add(val int) {
    q.data.PushBack(val)
}

func (q *Queue) Poll() int {
	elem := q.data.Front() // queue is first in, first out
	if elem == nil {
		return -1 // or whatever identifier that makes sense to the application, e.g. math.MinInt
    }
	
	value := q.data.Remove(elem)
	return value.(int)
}

func (q *Queue) Peek() int {
	elem := q.data.Front()
	return elem.Value.(int)
}

func (q *Queue) Len() int {
	return q.data.Len() // get the current number of elements in the queue
}

// what's more, we can print the current queue
func (q *Queue) Print() {
	for e := q.data.Front(); e != nil; e = e.Next() {
		fmt.Printf("%d ", e.Value)
    }
	fmt.Println()
}

func (q *Queue) Clear() {
	q.data.Init() // clear all data in the queue; basically re-initialized the list
}
```

There is one more convenient function `MoveToFront` that moves the specified element to the head of the list.
I find it really help when implementing an LRU cache. There is `MoveToBack` available too, if you prefer.

```go
type LRU struct {
	data *list.List
	m    map[string]*list.Element
	size int
}

func NewCache(capacity int) *LRU {
    return &LRU{
        data: list.New(),
        m:    make(map[string]*list.Element),
        size: capacity,
    }
}

func (c *LRU) Get(key string) any {
    elem, ok := c.m[key]
    if !ok {
        return nil
    }

    // move the last visited element to the head of the list
    c.data.MoveToFront(elem)
    return e.Value
}

func (c *LRU) Put(key string, val any) {
    if c.data.Len() >= c.size {
        c.evict()
    }

    elem := c.data.PushFront(val)
    c.m[key] = elem
}

func (c *LRU) evict() {
	elem := c.data.Back() // remove the least recently visited element from the tail of the list
    c.data.Remove(elem)
}
```

### container/heap

Go also provides an interface to construct min heap, which is an ordered complete binary tree where each node is smaller
than all its children.

```go
package heap

type Interface interface {
	sort.Interface
	Push(x any) // add x as element Len()
	Pop() any   // remove and return element Len() - 1.
}
```

And sort interface allows user to define sorting behaviours of custom collections.

```go
package sort

type Interface interface {
	// Returns the number of elements in the collection
	Len() int
	// Return true if element at index i is less than element at index j
	Less(i, j int) bool
    // Swap swaps the elements with indexes i and j.
	Swap(i, j int)
}
```

Now we have the arsenal to try to implement a `PriorityQueue` in Go.

```go
type Stock struct {
	Code  string
	Price int
}

type PriorityQueue struct {
    stocks []*Stock
}

// implement sort.Interface

func (q PriorityQueue) Less(i, j int) bool {
    return q.stocks[i].Price < q.stocks[j].Price
}

func (q PriorityQueue) Swap(i, j int) {
    q.stocks[i], q.stocks[j] = q.stocks[j], q.stocks[i]
}

func (q PriorityQueue) Len() int {
	return len(q.stocks)
}

// implement heap.Interface

func (q *PriorityQueue) Push(s any) {
    item := x.(*Stock)
    q.stocks = append(q.stocks, item)
}

func (q *PriorityQueue) Pop() any {
    size := len(q.stocks)
    if size == 0 {
        return nil
    }

    item := q.stocks[size-1]
    q.stocks = q.stocks[:size-1]

    return item
}
```

If we run through an example, we will see that the Stocks are sorted by their respective prices in ascending order.

```go
func main() {
    pq := PriorityQueue{
        stocks: make([]*Stock, 0),
    }

    list := []*Stock{
        {
            Code:  "GOOG",
            Price: 197,
        },
        {
            Code:  "SQ",
            Price: 91,
		},
        {
            Code:  "TSLA",
            Price: 462,
        }
    }

    for _, s := range list {
        pq.Push(s)
    }
    
    heap.Init(&pq) // build up the heap properties by heapifying the collection

	count := pq.Len()
    for i := 0; i < count; i++ {
        s := heap.Pop(&pq).(*Stock)
        fmt.Printf("code: %s, price: %d\n", s.Code, s.Price)
    }
}
```

Below output is seen as expected

```shell
code: SQ, price: 91
code: GOOG, price: 197
code: TSLA, price: 462
```

+++
date = '2023-11-29T11:17:04+08:00'
draft = false
title = 'Built-in functions in Go'
+++

The builtins are functions in Golang that you can call directly *without having to import any package*; they just come with the programming language itself.

The full list resides in the builtin standard library: `https://pkg.go.dev/builtin`

Many of the builtin functions are widely used, so it's important to know them by heart.

The motivation for this post comes from the fact that `Go 1.21.0` has introduced multiple new handy builtins (which doesn't happen frequently).

### min(), max()

The most exciting addition in `Go 1.21.0` in my opinion! Before that, there isn't out-of-box support for numeric comparison like `Math.min()` in Java.
As a result, I often ended up writing my own utility function to do it.

Now the builtins support to select the minimum/maximum out of one or more values we specified.

The functions take in integer, float (select in numeric order) and string (select in alphabetical order). The usage is quite self-explanatory:

```go
n := min(5, 1, -2, 4, 3)
fmt.Println(n) // -2

m := max(5, 1, -2, 4, 3)
fmt.Println(m) // 5

s := min("derek", "chris", "hardy", "Christopher")
fmt.Println(s) // Christopher
```


### len()

It returns the length of a data structure based on its type, which usually means the number of elements currently present in it.

To break down a bit further:

| Type                | Length                                               |
|---------------------|------------------------------------------------------|
| array / slice / map | number of elements contained                         |
| string              | number of bytes / characters (since 1 char = 1 byte) |
| channel             | number of elements in the buffer                     |

```go
func main() {
	data := []int{1, 2, 3, 4, 5}
	fmt.Println(len(data)) // outputs 5
}
```

There is another call `cap()` to get the capacity of the object, which means the total number of elements that can be held without resizing (usually doubling) the underlying allocated space.


### make()

It allocates memory and initialises the object (slice/map/channel/struct).

Optionally you can specify the length & capacity, beyond which a slice / map will resize by allocating more space dynamically to accommodate more elements.

```go
func main() {
	myMap := make(map[string]int)
    // specify the slice length to be 5, and capacity to be 10
	mySlice := make([]int, 5, 10)
	
	fmt.Println(len(myMap))   // 0
	fmt.Println(len(mySlice)) // 5
}
```

You may have already noticed, `make()` and composite literals are two ways in Go to initialise data structures.

Using composite literal allows you to provide *initial values*. For example:

```go
s := []int{1, 2}

m := map[string]int{
	"derek": 1,
	"hardy": 2,
}
```

Otherwise, they are pretty much the same. 

There is actually one more `new()` builtin, which allocates space for a given type and returns 
a pointer to the zero value of that type.


### append()

Specific to slice data structure, it literally appends given elements to the end of a slice.

The function then returns the updated slice.

```go
mySlice := make([]string, 0)
mySlice = append(mySlice, "one", "two", "three")
fmt.Println(len(mySlice)) // returns 3
```

### copy()

Specific to slice data structure, it copies elements from source slice to destination slice.

The function then returns the number of elements copied. It's useful if you need a deep copy of the slice.

```go
src := []int{1, 2, 3}
dest = make([]int, 3)

n := copy(src, dest) // n = 3, dest = []int{1,2,3}
```

### delete()

Specific to map data structure, it literally deletes a key-value pair from the map.

```go
myMap := map[string]int{
	"derek": 1,
	"hardy": 2,
}
fmt.Println(myMap) // map[derek:1 hardy:2]

delete(myMap, "derek")
fmt.Println(myMap) // map[hardy:2]
```

### clear()

It is newly added in `Go 1.21.0` that deletes all entries for map & slice.

If you are familiar with Java, it is the same as `clear()` method in HashMap / List.

```java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        Map<String, Integer> m = new HashMap<>();
        m.put("derek", 1);
        System.out.println(m); // {derek=1}
        
        m.clear();
        System.out.println(m); // {}
    }
}
```

In Go, it works as below:

```go
myMap := map[string]int{
	"derek": 1,
	"hardy": 2,
}
fmt.Println(myMap) // map[derek:1 hardy:2]

clear(myMap)
fmt.Println(myMap) // map[]
```

### panic() and recover()

Quote the explanation from official Go doc:

> The panic built-in function stops normal execution of the current goroutine

Usually we place it under the error path:

```go
resp, err := testFunc()
if err != nil {
	panic(err)
}
defer cleanUp()
```

The `defer` statements within the same calling context as `panic` will still be executed.

However, we seldom use this pattern in the production service code because error can be better handled and logged.
Furthermore, fallback strategy should be in place to degrade the service instead of ceasing it abruptly in most cases.

The `panic-recover` pair in Golang is similar to `try-catch` in Java. (`defer` for `finally` if you wish)

`recover()` has to be called in the `defer` statement (the only code that can execute after panic) to regain control and handle error passed into the `panic()` earlier on.

```go
func run() {
	defer func() { // define anonymous function
		if r := recover(); r != nil { // the call to recover returns the error passed in to panic
			fmt.Printf("RECOVERED FROM ERR: %v\n", r)
        }
    }() // trigger the anonymous function we just defined
	
	err := execute() // function defined elsewhere
	if err != nil {
		panic(err)
    }
}
```

Still I rarely use this feature. It might be simpler to enrich the error context (e.g. `fmt.Errorf("failed to run, %w", err)`) and pass the error up to the caller and handle at the
most appropriate level.
It also greatly improves the traceability of error and makes debugging a lot easier.

### print() and println()

I'm sure `fmt.Println("Hello World!")` is the first line in everyone's Go journey.

The builtin counterparts `print` and `println` do the same thing except output to the standard error.

*FYI*: the file descriptor for `stdout` is 1, and for `stderr` is 2. So the `fmt` and builtin actually output to different streams.

You usually want to separate regular application output (for observability) from error messages (for diagnostics). To do so, for example:

```bash
./my_program > out.txt 2> err.txt
```

It might come in handy to use builtin `print` in dev work for debugging to save some typings. 

However, it is still important to consider logger for observability in the production service.

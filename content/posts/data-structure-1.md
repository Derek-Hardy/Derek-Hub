+++
date = '2023-12-05T20:28:17+08:00'
draft = false
title = 'Data Structure in Go - Part 1'
+++

### Alias type

Data structures start with the fundamental data types in Go: 

`Numbers`, `String`, `Boolean`, `Array`, `Struct`, `Slice`, `Pointer`

Besides the basic ones in Go, there are alias types that just provide more meaningful names for existing data types.
It's good to be aware of them so as not to get confused when you happen to see them.

| Alias type | Underlying type | Why?                                      |
|------------|-----------------|-------------------------------------------|
| rune       | int32           | represent a Unicode character             |
| byte       | uint8           | represent one byte (8 bits) of data       |
| int        | int32 / int64   | adjusted for the system (32 bit / 64 bit) |

We can also define custom alias type:
```go
type MyNumber int32 // the alias MyNumber has underlying type as int32
```

### Receiver

When defining methods for a custom struct, we want to take note of the difference between *value receiver* and *pointer receiver*.

For a **value receiver**, the method operates on a copy of the receiver. So the below setter does not modify the original object.

I seldom use this pattern when designing system with object-oriented programming in mind. It might make more sense if the struct is read-only;
or write the code in a more functional style.

```go
type MyStruct struct {
	Data int32
}

// '(m MyStruct)' is the value receiver for the method 'Set'
func (m MyStruct) Set(data int32) {
	m.Data = data
}

func main() {
    s := MyStruct{Data: 5}
    s.Set(10)
    fmt.Println(s.Data) // 5, the setter didn't modify the object
}
```

For a **pointer receiver**, the method operates on the pointer to the receiver. In this case, any modification in the method affects the original object.

I think it's preferred in most cases when we want to maintain and update struct states.

```go
type MyStruct struct {
	Data int32
}

// '(m *MyStruct)' is the pointer receiver for the method 'Set'
func (m *MyStruct) Set(data int32) {
	m.Data = data
}

func main() {
	s := MyStruct{Data: 5}
	s.Set(10)
	fmt.Println(s.Data) // 10, the setter did modify the object
}
```

### Interface

In Go, interface is a type that holds values of a concrete type. Such a concrete type must implement all methods defined in the interface.

Unlike Java where interface is explicitly declared to be implemented:

```java
interface Api {
    void batchWrite(String id, List<String> data);
    List<String> read(String id);
}

class DataBase implements Api {
    private Map<String, List<String>> m = new HashMap<>();
    
    public void batchWrite(String id, List<String> data) {
        this.m.put(id, data);
    }
    
    public List<String> read(String id) {
        return this.m.get(id);
    }
}
```

A Go struct implicitly implements an interface, if it has implemented all required methods.

```go
type Api interface {
	BatchWrite(id string, data []string)
	Read(id string) []string
}

// DataBase has implemented Api interface with methods defined below
type DataBase struct {
	data map[string][]string
}

func (d *DataBase) BatchWrite(id string, data []string) {
	d.data[id] = data
}

func (d *DataBase) Read(id string) []string {
	return d.data[id]
}
```

When a method is called on an interface, Go *dynamically dispatch* the call to the right method implementation based on its stored type & value.

A special case is the *empty interface* - i.e. `interface{}`. Since no method is defined, all types can be considered to have implemented this interface.

The reverse is also true - `interface{}` can represent any type.

The Go interface stores two things at runtime:

1. a **type** - information about the dynamic type of the value it holds
2. a **value** - pointer to the actual data

To visualise this:

```go
func describe(i interface{}) {
	fmt.Printf("type: %T, value: %v\n", i, i)
}

func main() {
	var api Api
    api = &DataBase{
        data: make(map[string][]string),
    }
	
    describe(api) // type: *main.DataBase, value: &{map[]}
}
```


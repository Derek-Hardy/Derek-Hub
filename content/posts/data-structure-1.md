+++
date = '2024-12-12T20:28:17+08:00'
draft = false
title = 'Data Structure in Go'
+++

Data structures start with the fundamental data types in Go: 
`Numbers`, `String`, `Boolean`, `Array`, `Struct`, `Slice`, `Pointer`

Besides the basic ones in Go, there are alias types that just provide more meaningful names for existing data types.
It's good to take note of them so as not to get lost when you stumble into them.

| Alias type | Underlying type | Why?                                      |
|------------|-----------------|-------------------------------------------|
| rune       | int32           | represent a Unicode character             |
| byte       | uint8           | represent one byte (8 bits) of data       |
| int        | int32 / int64   | adjusted for the system (32 bit / 64 bit) |

We can also define custom alias type:
```go
type MyNumber int32
```



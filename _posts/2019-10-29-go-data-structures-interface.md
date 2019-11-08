---
layout:     post
title:      Go Interface explained
date:       2019-10-29 12:31:19
summary:    Usage and implementation of interface in Go
categories: go data-structure interface
---
Go's interfaces are like Frankenstein: for the static part, types are checked at compile time; as for dynamic, methods can be looked-up for at runtime. This post is MHO on how does it work and how it's implemented.


### How interfaces work

Interfaces fulfill [duck typing](https://en.wikipedia.org/wiki/Duck_typing) which let you use objects like you would in dynamic languages, yet still have the compiler catch obvious mistakes for you like calling method with wrong number or wrong type of arguments.

For example you have a function, taking a `ReadCloser`, calls `Read` repeatedly and then `Close` after getting all the data needed.
```go
func ReadThenClose(rc ReadCloser, buf []byte) (n int, err os.Error) {
    for len(buf) > 0 && err == nil {
        var nr int
        nr, err = rc.Read(buf)
        n += nr
        buf = buf[nr:]
    }
    rc.Close()
    return
}
```
`rc` can be any type as long as it has `Read` and `Close` methods with matching signatures as `ReadCloser` described:
```go
type ReadCloser interface {
    Read(b []byte) (n int, err os.Error)
    Close()
}
```
If you pass a value of a wrong type, you get an error at compile time, unlike dynamic languages such as Python and JavaScript which complain at run time.

Interfaces also allow you to lookup for methods dynamically. A typical example is that how to get the string representation of an object:
```go
type Stringer interface {
    String() string
}

func ToString(i interface{}) string {
    if s, ok := i.(Stringer); ok {
        return s.String()
    }
    switch s := i.(type) {
        case int:
            return strconv.Itoa(s)
        case float:
		    return strconv.FormatFloat(s, 'g', -1, 64)
    }
    return "???"
}
```
The arg `i` is of static type `interface{}` which guarantees no methods and could contain any type. The `if` statement queries whether it is possible to convert `i` to a value of type `Stringer` which has the method `String`.
If that is the case, the `String` method is called. Otherwise, the `switch` tries to pick up some basic types before giving up. This function is stripped down version of what the `fmt` package doses.

Here is another example, a type with a `String` method which returns the string representation of a 2D position and a trivial `Get` method.
```go
type index uint64

func (i index) String() string {
	return fmt.Sprintf("%d", uint64(i))
}

type Pos struct {
	x index
	y index
}

func (p Pos) String() string {
	x, y := p.Get()
	return fmt.Sprintf("(x:%d, y:%d)", x, y)
}

func (p Pos) Get() (uint64, uint64) {
	return uint64(p.x), uint64(p.y)
}
```
A value of type `Pos` can be passed to `ToString`, and its `String` method will be called to format a string to return. The runtime knows that `Pos` has a `String` method, so it **implements** the `Stringer` interface.

This example shows that implicit conversions are checked at compile time and explicit interface-to-interface conversions inquire about method sets at runtime.


### Interface value under the hood

Languages with methods typically fall into one of the two categories: method tables are prepared statically for method calls (like C/C++ and Java), or make a name lookup for each method call and use caching to make that call efficient (like Python and JavaScript). Go lies somewhere in-between: it has method tables but computes them at runtime.

A value of `Pos` contains two 64-bit word (we are assuming a 64-bit machine):

![](/images/go-data-structures-interface/PH_001.png)

A Interface value is composed of a 2-word pair holding one pointer to the information about the type stored in the interface and one pointer to the data associated. Assigning `p` to an interface value of `Stringer` type sets both words of the interface.

![](/images/go-data-structures-interface/PH_002.png)

The first word in the interface value points to an interface table, or itable. The itable starts with metadata about the type involved followed by a list of function pointers. Note that the itable corresponds to the **interface type**, not the **dynamic type**. In the example above, the itable for `Stringer` contains the methods used to satisfy `Stringer` which is just **Pos's** String, and Pos's other method (Get) make no appearance in the itable.

The second word in the interface value points to the actual data, in the case above a copy of `p`. The assignment `var s Stringer = p` makes a copy of `p`: if b later changes, `s` is supposed to have the original value, not the new one. Values stored in interfaces might be large, but only one word is used to hold the value in the interface structure, so the assignment allocates a chunk of memory on the heap and records the pointer in the one-word slot.

To check whether an interface value holds a particular type, as in the type switch above, the compiler generates code equivalent to the C expression `s.tab->type` to obtain the type pointer and check it against the desired type. If the types match, the value can be copied by dereferencing `s.data`.

To call s.String(), the Go compiler generates code that equivalent to the C expression `s.tab->fun[0](s.data)`. It calls the appropriate function pointer from the itable, passing the interface value's data word as the function's first argument. Note that the function in the itable is being passed the pointer from the second word of the interface value, not the value it points at. In general, the interface doesn't know the meaning of this word nor how much data it points at. Instead, the interface code arranges that the function pointers in the itable expect the 64-bit representation stored in the interface values. Thus the function pointer in this example is `(*Pos).String` instead of `Pos.String`.


### Method table computing

Itables are associated with one pair of interface type and one concrete type, and there are too many interface type and concrete type paris of which most are not needed. Instead of precomputing all itables at compile time, compiler generates a type-description structure for each concrete type like `Pos` or `func(map[string]int)`, containing a list of methods implemented by that type. Similarly, compiler generates a type-description structure (different from the former) for each interface type, also containing a method list. The runtime computes the itable by looking up each method listed in the interface type's method list in the concrete type's method list. The itable was cached by the runtime after generated, so it will only be computed once.

In the previous simple example, `Pos` has two methods and `Stringer` has only one. In general, a concrete type may have *n* methods and interface type has *m*. The search to find mapping from interface methods to concrete methods has time complexity of *O(n * m)*. By sorting the two method tables and walking them simultaneously, the time cost will be reduced to *O(n + m)*.


### Memory Optimization

If the interface type involved is empty, containing no methods, then the itable serves no purpose else but holding the pointer to the concrete type. In that case, the itable can be dropped and the first word will point to the type directly.

![](/images/go-data-structures-interface/PH_003.png)

If the value associated with the interface fit in one single word, there is no need to use indirection. The value will be stored in the second word of the interface value.

![](/images/go-data-structures-interface/PH_004.png)

Empty interfaces holding word-sized values can use both optimizations.

![](/images/go-data-structures-interface/PH_005.png)


---
layout:     post
title:      Go Interface explained
date:       2019-10-29 12:31:19
summary:    Usage and implementation of interface in Go
categories: go data-structure interface
---
Go's interfaces are like Frankenstein: for the static part, types are checked at compile time; as for dynamic, methods can be looked-up for at runtime. This post is MHO on how does it work and how it's implemented.

# How interfaces work
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
If that is the case, the `String` method is called to obtain a string. Otherwise, the `switch` tries to pick up some basic types before giving up. This function is stripped down version of what the `fmt` package doses.


Here is another example, a type with a `String` method which returns the string representation of a 64-bit integer in binary format.
```go
type Bin64 uint64

func (b64 Bin64) String() string {
    return strconv.FormatUint(b64.Get(), 2)
}

func (b64 Bin64) Get() uint64 {
    return uint64(b64)
}
```
A value of type `Bin64` can be passed to `ToString`, and its `String` method will be called to format a string to return. The runtime knows that `Bin64` has a `String` method, so it implements the `Stringer` interface.

This example shows that implicit conversions are checked at compile time and explicit interface-to-interface conversions inquire about method sets at runtime.
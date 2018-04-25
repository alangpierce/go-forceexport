# go-forceexport

go-forceexport is a golang package that allows access to any module-level
function, even ones that are not exported. You give it the string name of a
function , like `"time.now"`, and gives you a function value that calls that
function. More generally, it can be used to achieve something like reflection on
top-level functions, whereas the `reflect` package only lets you access methods
by name.

As you might expect, this library is **unsafe** and **fragile** and probably
shouldn't be used in production. See "Use cases and pitfalls" below.

It has only been tested on Mac OS X with Go 1.6 and on GNU/Linux with
Go 1.10rc2.
If you find that it works or breaks on other platforms, feel free to submit
a pull request with a fix and/or an update to this paragraph.

## Installation

`$ go get github.com/alangpierce/go-forceexport`

## Usage

Here's how you can grab the `time.now` function, defined as
`func now() (sec int64, nsec int32)`

```go
var timeNow func() (int64, int32)
err := forceexport.GetFunc(&timeNow, "time.now")
if err != nil {
    // Handle errors if you care about name possibly being invalid.
}
// Calls the actual time.now function.
sec, nsec := timeNow()
```

The string you give should be the fully-qualified name. For example, here's
`GetFunc` getting itself.

```go
var getFunc func(interface{}, string) error
GetFunc(&getFunc, "github.com/alangpierce/go-forceexport.GetFunc")
```

## Use cases and pitfalls

This library is most useful for development and hack projects. For example, you
might use it to track down why the standard library isn't behaving as you
expect, or you might use it to try out a standard library function to see if it
works, then later factor the code to be less fragile. You could also try using
it in production; just make sure you're aware of the risks.

There are lots of things to watch out for and ways to shoot yourself in
the foot:
* If you define the wrong function type, you'll get a function with undefined
  behavior that will likely cause a runtime panic. The library makes no attempt
  to warn you in this case.
* Calling unexported functions is inherently fragile because the function won't
  have any stability guarantees.
* The implementation relies on the details of internal Go data structures, so
  later versions of Go might break this library.
* Since the compiler doesn't expect unexported symbols to be used, it might not
  create them at all, for example due to inlining or dead code analysis. This
  means that functions may not show up like you expect, and new versions of the
  compiler may cause functions to suddenly disappear.
* If the function you want to use relies on unexported types, you won't be able
  to trivially use it. However, you can sometimes work around this by defining
  equivalent copies of those types that you can use, but that approach has its
  own set of dangers.

## How it works

The [code](/forceexport.go) is pretty short, so you could just read it, but
here's a friendlier explanation:

The code uses the `go:linkname` compiler directive to get access to the
`runtime.firstmoduledata` symbol, which is an internal data structure created by
the linker that's used by functions like `runtime.FuncForPC`. (Using
`go:linkname` is an alternate way to access unexported functions/values, but it
has other gotchas and can't be used dynamically.)

Similar to the implementation of `runtime.FuncForPC`, the code walks the
function definitions until it finds one with a matching name, then gets its code
pointer.

From there, it creates a function object from the code pointer by calling
`reflect.MakeFunc` and using `unsafe.Pointer` to swap out the function object's
code pointer with the desired one.

Needless to say, it's a scary hack, but it seems to work!

## License

MIT
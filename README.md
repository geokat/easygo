# (Not so) Easy Go

Go is a wonderful language. It's easy to learn and a pleasure to use;
it has a great balance of flexibility and rigidity that allows me to be
productive without feeling like I'm building on quicksand.

But, like any other language, it has its share of edge cases,
idiosyncrasies and even inconsistencies that can leave you scratching
your head. The following is an attempt to catalog such bits of Go,
along with an attempt to explain them, so that I can understand them
better.

## Table of Contents

- [Slices](#slices)
  - [Underlying arrays matter](#underlying-arrays-matter)
  - [Slicing direction matters](#slicing-direction-matters)
- [Arrays](#arrays)
  - [Array's length is part of its type](#arrays-length-is-part-of-its-type)
  - [Arrays are value types](#arrays-are-value-types)

## Slices

### Underlying arrays matter

```Go
package main

import "fmt"

func main() {
	s1 := []int{0, 1, 2}
	// Prints 3 (the size of the underlying array)
	fmt.Println(cap(s1))

	s2 := s1[0:2]
	// Prints 3 (s2 has the same underlying array as s1)
	fmt.Println(cap(s2))

	s2[0] = -1
	// Prints -1 (s2 has the same underlying array as s1)
	fmt.Println(s1[0])

	s2 = append(s2, 3)
	// Prints 3 (s2 still has the same underlying array as s2)
	fmt.Println(s1[2])

	s2 = append(s2, 4)
	s2[0] = 0
	// Prints -1 (s1 and s2 don't share underlying array anymore)
	fmt.Println(s1[0])
}
```

[Go Playground link](https://go.dev/play/p/3OQrnWUJ2Nk)

The last append (`s2 = append(s2, 4)`) takes `s2` over its capacity,
so Go creates a new underlying array for it.

### Slicing direction matters

```Go
package main

import (
	"fmt"
)

func main() {
	s1 := []int{0, 1, 2, 4}
	// Prints 4
	fmt.Println(cap(s1))

	s2 := s1[:2]
	// Prints 4
	fmt.Println(cap(s2))

	s2 = s2[:4]
	// Prints 4 (re-slicing upwards within capacity works)
	fmt.Println(s2[3])

	s2 = s1[1:]
	// Prints 3 (slicing off the bottom changes capacity)
	fmt.Println(cap(s2))

	s2 = s2[1:]
	// Prints 2
	fmt.Println(cap(s2))
}
```

[Go Playground link](https://go.dev/play/p/WKCjvUCIQb4)


Slicing off the top of a slice doesn't change its capacity and we can
re-slice back up to get the original length (and elements). Slicing
off the bottom, however, changes the capacity (i.e. there's no way to get
the sliced off elements back).

## Arrays

Arrays are rarely used explicitly in Go. Usually you just create a
slice where other languages may use arrays. Go implicitly creates an
underlying array and takes care of deallocating it once the slice
stops using it.

### Array's length is part of its type

```Go
package main

func main() {
	var a1 [2]int
	var a2 [2]int
	var a3 [3]int
	// Works
	a1 = a2
	// Causes a compile-time error
	a1 = a3

	_ = a1
}
```

[Go Playground link](https://go.dev/play/p/Sp1u1tpqoPr)

`a1 = a3` does not fall within any of the
[assignability](https://go.dev/ref/spec#Assignability) rules: `a1` and
`a3` have different underlying types, since an array's length is part
of its type.

### Arrays are value types

```Go
package main

import "fmt"

func main() {
	a1 := [...]int{0, 1, 2, 3}

	a2 := a1

	a2[0] = 1

	// Prints 0
	fmt.Println(a1[0])
}
```

[Go Playground link](https://go.dev/play/p/5ZCIjKWPtnD)


Unlike slices, arrays are value types, which means that passing an array
actually copies all of it, including the memory that holds the data.

## Interfaces

### Nil inside an interface is not equal to nil

```Go
package main

import "fmt"

type MyError struct{}

// Implements the error interface
func (me MyError) Error() string {
	return "my error"
}

func main() {
	var myErr *MyError
	var err error = myErr

	// Prints true
	fmt.Println(myErr == nil)

	// Prints false
	fmt.Println(err == nil)

	// Prints true
	fmt.Println(err.(*MyError) == nil)
}
```

[Go Playground link](https://go.dev/play/p/9GIRa8UxdzA)

[Explained](https://go.dev/doc/faq#nil_error) in the Go FAQ.

Also, [this post](https://groups.google.com/g/golang-nuts/c/wnH302gBa4I/m/zqcfBFPIBgAJ)
by a Go developer provides more context:
> ... we, probably incorrectly, chose to use the same identifier, `nil`, to
> represent the zero value both for pointers and for interfaces.
> ...
> If we do change something in this area for Go 2, which we probably
> won't, my preference would definitely be to introduce `nilinterface`
> rather than to further confuse the meaning of `r != nil`.

## Assignability Rules

### Numeric types are defined types

```Go
package main

func main() {
	type MyMap1 map[int]int
	type MyMap2 map[int]int

	var m1 MyMap1
	var m2 MyMap2
	var m map[int]int

	// Works
	m1 = m

	// Causes compile-time error
	m1 = m2

	type MyInt int

	var d1 MyInt
	var d2 int

	// Causes compile-time error
	d1 = d2

	// Avoid "declared but not used" errors
	_, _ = m1, d1
}
```

[Go Playground link](https://go.dev/play/p/DAJzL9o83cI)

`m1 = m` works because it falls within the
[assignability](https://go.dev/ref/spec#Assignability) rules. In this
case, they both have identical underlying types and at least one of
them (`m`) does not have a [named type](https://go.dev/ref/spec#Types).

`m1 = m2` causes a compilation error despite the identical underlying
types, because they both have defined types.

Finally, `d1 = d2` causes an error because `int` is also a named
type—predeclared types are considered named types according to
the spec, which also mentions this:

> To avoid portability issues all numeric types are defined types...

## Defer and Recover

### Defer statements can't change return parameters?

```Go
package main

import (
	"fmt"
)

func main() {
	// Prints foo
	fmt.Println(testFunc())
}

func testFunc() string {
	result := "foo"
	defer func() {
		result = "bar"
	}()

	return result
}
```

[Go Playground link](https://go.dev/play/p/bEsl6FVDuo3)

According to the [spec](https://go.dev/ref/spec#Defer_statements):

> ... if the surrounding function returns through an explicit return
> statement, deferred functions are executed after any result
> parameters are set by that return statement but before the function
> returns to its caller

However:

> ... if the deferred function is a function literal and the
> surrounding function has named result parameters that are in scope
> within the literal, the deferred function may access and modify the
> result parameters before they are returned

Which means that the following code works as expected:

```Go
package main

import (
	"fmt"
)

func main() {
	// Prints bar
	fmt.Println(testFunc())
}

func testFunc() (result string) {
	result = "foo"
	defer func() {
		result = "bar"
	}()

	return result
}
```

[Go Playground link](https://go.dev/play/p/NuJJ5O-ISny)

### `recover()` won't save from panic in spawned goroutine

```Go
package main

import (
	"fmt"
	"sync"
)

func main() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Println("Caught:", err)
		}
	}()

	var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		defer wg.Done()
		// Won't be caught
		panic("spawned goroutine panic")
	}()

	wg.Wait()
}
```
[Go Playground link](https://go.dev/play/p/ffZv-NqHvJj)

`recover()` has to be called on the panicking goroutine in order to
stop the panicking. This one can be easy to miss, but it follows
directly from the [spec](https://go.dev/ref/spec#Handling_panics).

## Package sync

### Read-preferring or write-preferring mutex?

With reader/writer mutexes, there are [two opposing
priority](https://en.wikipedia.org/wiki/Readers%E2%80%93writer_lock#Priority_policies)
policies.  Which one does `sync.RWMutex` implement?

According to the [doc](https://pkg.go.dev/sync#RWMutex):

> If any goroutine calls Lock while the lock is already held by one or
> more readers, concurrent calls to RLock will block until the writer
> has acquired (and released) the lock, to ensure that the lock
> eventually becomes available to the writer.

However, at the time of this writing (go1.22), it doesn't appear to be
the whole story. Consider the following example:

```Go
package main

import (
	"fmt"
	"sync"
	"time"
)

const (
	write = "write"
	read  = "read"
)

var mtx sync.RWMutex
var wg sync.WaitGroup

func runWorker(num int, mode string) {
	go func() {
		// Order lock requests by their consecutive number.
		time.Sleep(time.Duration(num*10) * time.Millisecond)

		fmt.Printf("worker %d requesting lock (for %s)\n", num, mode)

		if mode == write {
			mtx.Lock()
		} else {
			mtx.RLock()
		}

		fmt.Printf("worker %d acquired lock\n", num)

		time.Sleep(100 * time.Millisecond)

		if mode == write {
			mtx.Unlock()
		} else {
			mtx.RUnlock()
		}

		fmt.Printf("worker %d released lock\n", num)
		wg.Done()
	}()
}

func main() {
	wg.Add(4)
	runWorker(1, read)
	runWorker(2, write)
	runWorker(3, write)
	runWorker(4, read)
	wg.Wait()
}
```

[Go Playground link](https://go.dev/play/p/2wwHelKprZX)

The output is:

```
worker 1 requesting lock (for read)
worker 1 acquired lock
worker 2 requesting lock (for write)
worker 3 requesting lock (for write)
worker 4 requesting lock (for read)
worker 1 released lock
worker 2 acquired lock
worker 2 released lock
worker 4 acquired lock
worker 4 released lock
worker 3 acquired lock
worker 3 released lock
```

Indeed, the first blocked write lock request (in worker 2) results in
the following read request getting blocked too, despite the fact that
the mutex is locked for reading. And once the lock is released, the
writer acquires it before the reader. This would appear to confirm
that `sync.RWMutex` is write-preferring.

However, when the writer is done with the mutex, the remaining blocked
writer takes the back seat to the blocked reader (i.e. the reader gets
to acquire the lock first).

This suggests that `sync.RWMutex` is *write-preferring when locked for
reading and read-preferring when locked for writing*.

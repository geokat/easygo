# (Not so) Easy Go

Go is a wonderful language. It's easy to learn and a pleasure to use;
it has a great balance of flexibility and rigidity that allows me to be
productive without feeling like I'm building on quicksand.

But, like any other language, it has its share of edge cases,
idiosyncrasies and even inconsistencies that can leave you scratching
your head. The following is an attempt to catalog such bits of Go,
along with an attempt to explain them, so that I can understand them
better.

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

Slicing off the top of a slice doesn't change its capacity and we can
re-slice back up to get the original length (and elements). However,
slicing off the botom changes the capacity (i.e. there's no way to get
the sliced off elements back).

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

[Explained](https://go.dev/doc/faq#nil_error) in the Go FAQ.

Also, [this post](https://groups.google.com/g/golang-nuts/c/wnH302gBa4I/m/zqcfBFPIBgAJ)
by a Go developer provides more context:
> ... we, probably incorrectly, chose to use the same identifier, `nil`, to
> represent the zero value both for pointers and for interfaces.
> ...
> If we do change something in this area for Go 2, which we probably
> won't, my preference would definitely be to introduce `nilinterface`
> rather than to further confuse the meaning of `r != nil`.

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

### Underlying Arrays Matter

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
	// Prints -1 (s1 and s2 don't share underlying array anymore).
	fmt.Println(s1[0])
}
```
The last append (`s2 = append(s2, 4)`) takes `s2` over its capacity,
so Go creates a new underlying array for it.

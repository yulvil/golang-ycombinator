# golang-ycombinator

Notes I took while watching [Y Not - Adventures in Functional Programming by Jim Weirich](https://www.youtube.com/watch?v=FITJMJjASUs) (Ruby Conf 2012)

Implementation in golang

## Fixpoints

`sqrt(1) = 1`

`sqrt(0) = 0`

## Higher-Order Functions

http://play.golang.org/p/cgVhg-lPaM

```golang
package main

import "fmt"

type fii func(int) int

func makeAdder(x int) fii {
	return func(n int) int { return n + x }
}

func compose(f, g fii) fii {
	return func(n int) int { return f(g(n)) }
}

func add1(n int) int {
	return n + 1
}

func mul3(n int) int {
	return n * 3
}

func main() {
	fmt.Println(add1(10))
	fmt.Println(mul3(10))
	fmt.Println(mul3(add1(10)))
	fmt.Println(makeAdder(1)(10))
	fmt.Println(compose(mul3, add1)(10))
}
```

## Functional Refactoring

#### Tennent Correspondance Principle

http://play.golang.org/p/4FOX-ck1xa

```golang
package main

import "fmt"

// Tennent's Correspondence Principle
// expr == (():expr}()
// { stmt; ... } == ((){ stmt; ... })()
func main() {
	var n = 10
	fmt.Println(n == func() int { return n }())
	fmt.Println(mul3(10))
	fmt.Println(newmul3(10))
}

func mul3(n int) int {
	return n * 3
}

func newmul3(n int) int {
	return func() int { return n * 3 }()
}
```

#### Introduce Binding

http://play.golang.org/p/X956U7wSeC

```golang
package main

import "fmt"

// Introduce Binding
// Add an argument and pass a dummy value
// It should have no effect (argument unused)
func main() {
	var n = 10
	fmt.Println(n == func(z int) int { return n }(123))
	fmt.Println(mul3(10))
	fmt.Println(newmul3(10))
}

func mul3(n int) int {
	return n * 3
}

func newmul3(n int) int {
	return func(m int) int { return n * 3 }(1919)
}
```

#### Function Wrap

http://play.golang.org/p/zjqf5U__Tx

```golang
package main

import "fmt"

// Wrap Function
// You can wrap a function with one argument, e.g. "f(n int) int"
// inside a "lambda (z int) int" and call that lambda immediately
// with the same argument n.
// It should have no effect
func main() {
	var n = 10
	fmt.Println(n == func(z int) int { return n }(123))
	fmt.Println(makeAdder(1)(10))
	fmt.Println(newMakeAdder(1)(10))
}

type fii func(int) int

func makeAdder(x int) fii {
	return func(n int) int { return n + x }
}

func newMakeAdder(x int) fii {
	return func(v int) int { return func(n int) int { return n + x }(v) }
}
```

#### Inline Function

http://play.golang.org/p/uRTGLbHSsi

```golang
package main

import "fmt"

// Inline definition
func main() {
	// No inline definition
	fmt.Println(compose(mul3, add1)(10))

	//	fmt.Println(compose(mul3, func(n int) int { return n + 1 })(10))
	//	fmt.Println(compose(func(n int) int { return n * 3 }, func(n int) int { return n + 1 })(10))

	// Replace add1 by its definition
	fmt.Println(compose(mul3, func(x int) fii { return func(v int) int { return func(n int) int { return n + x }(v) } }(1))(10))

	// Replace mul3 by its definition
	fmt.Println(compose(func(n int) int { return func(z int) int { return n * 3 }(1234567) }, func(x int) fii { return func(v int) int { return func(n int) int { return n + x }(v) } }(1))(10))

	// Replace compose by its definition
	fmt.Println(func(f, g fii) fii { return func(n int) int { return f(g(n)) } }(func(n int) int { return func(z int) int { return n * 3 }(1234567) }, func(x int) fii { return func(v int) int { return func(n int) int { return n + x }(v) } }(1))(10))

	// Pure lambda expression
	// - no assignments
}

type fii func(int) int

func compose(f, g fii) fii {
	return func(n int) int { return f(g(n)) }
}

var add1 = makeAdder(1)

func mul3(n int) int {
	return func(z int) int { return n * 3 }(1234567)
}

func makeAdder(x int) fii {
	return func(v int) int { return func(n int) int { return n + x }(v) }
}
```

## Recursion

Problem: How do you call a function that is not defined yet?

This program does not compile: "undefined: fact"
```golang
package main

import "fmt"

func main() {
	var fact = func(n int) int {
		if n == 0 {
			return 1
		} else {
			return fact(n - 1)
		}
	}

	fmt.Println(fact(5))
}
```

http://play.golang.org/p/Eh_USaeROk

```golang
package main

import "fmt"

type fii func(uint64) uint64
type frr func(frr) fii

func main() {
	var factImprover = func(partial fii) fii {
		return func(n uint64) uint64 {
			if n == 0 {
				return 1
			} else {
				return n * partial(n-1)
			}
		}
	}

	var ferr = func(n uint64) uint64 { panic("SHOULD NEVER BE CALLED") }

	fmt.Println("F0")
	var f0 = factImprover(ferr)
	fmt.Println(f0(0))
	// fmt.Println(f0(1)) // Does not work - panic

	fmt.Println("F1")
	var f1 = factImprover(f0)
	fmt.Println(f1(0))
	fmt.Println(f1(1))
	// fmt.Println(f1(2)) // Does not work - panic

	fmt.Println("F2")
	var f2 = factImprover(f1)
	fmt.Println(f2(0))
	fmt.Println(f2(1))
	fmt.Println(f2(2))
	// fmt.Println(f2(3)) // Does not work - panic

	fmt.Println("FX")
	var fx = factImprover(factImprover(ferr))
	fmt.Println(fx(0))
	fmt.Println(fx(1))
	//fmt.Println(fx(2))
	// fmt.Println(fx(3)) // Does not work - panic

	fmt.Println("FX2")
	var fx2 = func(improver func(z fii) fii) fii {
		return improver(improver(improver(improver(ferr))))
	}(factImprover)
	fmt.Println(fx2(0))
	fmt.Println(fx2(1))
	fmt.Println(fx2(2))
	fmt.Println(fx2(3))
	// fmt.Println(fx2(4)) // Does not work - panic

	fmt.Println("FX3")
	var fx3 = func(improver func(frr) fii) fii {
		return improver(improver)
	}(func(partial frr) fii {
		return func(n uint64) uint64 {
			if n == 0 {
				return 1
			} else {
				return n * (partial(partial))(n-1)
			}
		}
	})
	fmt.Println(fx3(0))
	fmt.Println(fx3(1))
	fmt.Println(fx3(2))
	fmt.Println(fx3(3))
	fmt.Println(fx3(10))
	fmt.Println(fx3(20))
	// ...
}
```

## Y-Combinator

http://play.golang.org/p/UphOerYr3V

```golang
package main

import "fmt"

type fii func(uint64) uint64
type frr func(frr) fii

func main() {
	//var ferr = func(n uint64) uint64 { panic("SHOULD NEVER BE CALLED") }

	fmt.Println("FX")
	var fx = func(gen func(frr) fii) fii {
		return gen(gen)
	}(func(gen frr) fii {
		return func(code fii) fii {
			return func(n uint64) uint64 {
				if n == 0 {
					return 1
				} else {
					return n * (gen(gen))(n-1)
				}
			}
		}(func(v uint64) uint64 { return (gen(gen))(v) })
	})
	fmt.Println(fx(20))

	fmt.Println("FX2")
	var fx2 = func(gen func(frr) fii) fii {
		return gen(gen)
	}(func(gen frr) fii {
		return func(partial fii) fii {
			return func(n uint64) uint64 {
				if n == 0 {
					return 1
				} else {
					return n * partial(n-1)
				}
			}
		}(func(v uint64) uint64 { return (gen(gen))(v) })
	})
	fmt.Println(fx2(20))

	// Extract the factorial func and pass it as a parameter
	fmt.Println("FX3")
	var fx3 = func(improver func(fii) fii) fii {
		return func(gen func(frr) fii) fii {
			return gen(gen)
		}(func(gen frr) fii {
			return improver(func(v uint64) uint64 { return (gen(gen))(v) })
		})
	}(func(partial fii) fii {
		return func(n uint64) uint64 {
			if n == 0 {
				return 1
			} else {
				return n * partial(n-1)
			}
		}
	})
	fmt.Println(fx3(20))

	// Split it in small pieces
	fmt.Println("FX4")
	var fact_improver = func(partial fii) fii {
		return func(n uint64) uint64 {
			if n == 0 {
				return 1
			} else {
				return n * partial(n-1)
			}
		}
	}

	// Applicative Order Y Combinator
	// aka Z-combinator
	// aka Fixpoint combinator
	var y = func(improver func(fii) fii) fii {
		return func(gen func(frr) fii) fii {
			return gen(gen)
		}(func(gen frr) fii {
			return improver(func(v uint64) uint64 { return (gen(gen))(v) })
		})
	}

	// Y calculates the fixpoint of an improver function
	var fact = y(fact_improver)
	fmt.Println(fact(20))

	// fact is the fixpoint of fact_improver
	fmt.Println(fact_improver(fact)(20))

}
```

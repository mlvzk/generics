# A proof-of-concept implementation of my generics proposal for Go

This program translates a Go file that uses generics into a regular Go file that can be run.

```
$ go get github.com/faiface/generics
```

Then navigate to the repo folder and run:

```
$ go install
```

This will install the `generics` command and you should be able to use it just by typing its name (if you have your [`$PATH` set up correctly](https://golang.org/doc/code.html)).

I have taken measures to prevent you from running this in production. Please, do **not** run this in production. The single measure taken is that this program only translates a single file. This means that generic functions and types are only usable within that one file.

Here's a trivial example.

```go
// reverse.go

package main

import "fmt"

func Reverse(a []type T) {
    for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
        a[i], a[j] = a[j], a[i]
    }
}

func main() {
    a := []int{1, 2, 3, 4, 5}
    b := []string{"A", "B", "C"}
    Reverse(a)
    Reverse(b)
    fmt.Println(a)
    fmt.Println(b)
}
```

Here we have a file called `reverse.go` that uses generics. Here's how we translate it:

```
$ generics -out out.go reverse.go
```

And here's what we get!

```go
package main

import "fmt"

func Reverse_int(a []int) {
    for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
        a[i], a[j] = a[j], a[i]
    }
}

func Reverse_string(a []string) {
    for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
        a[i], a[j] = a[j], a[i]
    }
}

func main() {
    a := []int{1, 2, 3, 4, 5}
    b := []string{"A", "B", "C"}
    Reverse_int(a)
    Reverse_string(b)
    fmt.Println(a)
    fmt.Println(b)
}
```

Then, of course, we can run `out.go`:

```
$ go run out.go
[5 4 3 2 1]
[C B A]
```

## More example

That was just a silly little example. For more complex examples, take a look into the [`examples`](examples/) directory:
- [Slice utilities](examples/sliceutils)
- [Channel utilities](examples/chanutils)
- [Math utilities](examples/mathutils)
- [Linked list](examples/list)
- [Sync map](examples/syncmap)

## The proposal

This is a refined version of a proposal I submitted a few weeks ago. [You can find the original version here.](https://gist.github.com/faiface/e5f035f46e88e96231c670abf8cab63f)

This version is very similar to the original proposal, it only differs in three things:
1. The `gen` keyword has been replaced with two keywords: `type` and `const`. This implementation only implements the `type` keyword, `const` will be described below nonetheless.
2. An `ord` type restriction in addition to the previously described `eq` and `num`.
3. The `type` keyword now must also appear in the declarations of generic types.

Now I will describe the proposal as concisely as I can. If you have questions, scroll down to the [FAQ](#FAQ) section.

### The `type` keyword

Let's start with generic functions. Here's a **pseudocode** of a generic `Map` function on slices:

```go
// PSEUDOCODE!!
func Map(a []T, f func(T) U) []U {
    result := make([]U, len(a))
    for i := range a {
        result[i] = f(a[i])
    }
    return result
}
```

> In case you don't know, a `Map` function takes a slice and a function and returns a new slice with each element replaced by the result of the function applied to the original element.
>
> For example: `Map([]float64{1, 4, 9, 16}, math.Sqrt)` returns a new slice `[]float64{1, 2, 3, 4}`, taking the square root of each of the original numbers.

To make this a **valid** Go code under my proposal, all you need to do is to mark the _first_ (and only the first) occurrence of each type parameter (= an unknown type) in the signature with the `type` keyword. The `Map` function has two:

```go
//           here              here
//            \/                \/
func Map(a []type T, f func(T) type U) []U {
    result := make([]U, len(a))
    for i := range a {
        result[i] = f(a[i])
    }
    return result
}
```

Nothing else changed.

The `type` keyword basically declares a type parameter in a signature. The name is then visible in the entire scope of the function.

There are three rules about the placement of the `type` keyword in signatures:
1. It's only allowed in package-level function declarations.
2. In functions, it's only allowed in the list of parameters. Particularly, it's **disallowed** in the list of results.
3. In methods, it's only allowed in the receiver type.

The last two rules can be remembered together: `type` is only allowed inside the first pair of parentheses.

### Unnamed type parameters

Okay, so no `type` in the list of results. But how do we make a function like this `Read`? The only occurrence of the `T` type is in the result:

```go
// DISALLOWED!!
func Read() type T {
    var x T
    fmt.Scan(&x)
    return x
}
```

To make this function work, we need to use an _unnamed type parameter_. It's basically a dummy generic parameter:

```go
func Read(type T) T {
    var x T
    fmt.Scan(&x)
    return x
}
```

The value of the unnamed parameter is irrelevant. We're only interested in the type. That's why when calling the `Read` function, we pass in the type directly:

```go
func main() {
    name := Read(string)
    age := Read(int)
    fmt.Printf("%s is %d years old.", name, age)
}
```

Don't worry, this doesn't send us to the [dependent typing land](https://en.wikipedia.org/wiki/Dependent_type) because we can't return types, only accept them.

It's simple, if a parameter is an unnamed generic parameter, you pass a type directly. Otherwise, you pass a value and the type system will infer the type.

This notation also makes it possible to give a type to the built-in `new` function:

```go
func new(type T) *T {
    var x T
    return &x
}
```

**One important rule:** generic functions cannot be used as values. They can't be assigned to variables and they can't be passed as arguments. To pass a specialized version of a generic function as an argument, wrap it in an anonymous function, like this:

```go
SomeFunction(func() int {
    return Read(int)
})
```

### Restricting types

Some functions (or types) want to declare that they don't work with all types, but only with ones that satisfy some conditions. For example, the keys of a map must be comparable. That is a restriction. A `Min` function only works on types that are orderable (i.e. can be compared with `<`).

Initially, my proposal excluded any support for restricting types for the purpose of simplicity. The [contracts proposal](https://go.googlesource.com/proposal/+/master/design/go2draft-contracts.md) by the Go team has received (justifiable) criticism for introducing complexity by supporting _contracts_, which make it possible to specify arbitrary restrictions on types.

But some restrictions are extremely useful. That's why I eventually decided to include three possible restrictions that should cover the majority of use-cases. This decision is governed by the [80/20 principle](https://en.wikipedia.org/wiki/Pareto_principle).

Here are the three possible restrictions:
1. **`eq`** - Comparable with `==` and `!=`. Usable as map keys.
2. **`ord`** - Comparable with `<`, `>`, `<=`, `>=`, `==`, `!=`. A subset of `eq`.
3. **`num`** - All numeric types: `int*`, `uint*`, `float*`, and `complex*`. Operators `+`, `-`, `*`, `/`, `==`, `!=`, and converting from untyped integer constants works. Not a subset of `ord`.

To use a type restriction, place it right after the first occurrence of the type parameter.

For example, here's the generic `Min` function:

```go
//                   here
//                    v
func Min(x, y type T ord) T {
    if x < y {
        return x
    }
    return y
}
```

Notice that `num` is not a subset of `ord`. This is because the complex number types are not comparable with `<`. To accept only the numeric types that are also orderable, combine the two restrictions like this: `type T ord num`.

The `eq`, `ord`, and `num` words have no special meaning outside of the generic definitions. They are not keywords.

### Generic types

We've covered everything about generic functions, let's move on to generic types.

To define a generic type, simply list the type parameters in parentheses right after the type name. Like this:

```go
// List is a generic singly-linked list.
type List(type T) struct {
    First T
    Rest  *List(T)
}
```

And as you can already see in the definition, to use a generic type, list the arguments in parentheses after the type name. For example `List(int)` is a list of integers, and `List(string)` is a list of strings.

When defining a type with multiple generic parameters, mark each one with `type`:

```go
// SyncMap is a generic hash-map usable from multiple goroutines simultaneously.
type SyncMap(type K eq, type V) struct {
    mu sync.Mutex
    m  map[K]V
}
```

Methods work as usual:

```go
func (sm *SyncMap(type K eq, type V)) Store(key K, value V) {
    sm.mu.Lock()
    sm.m[key] = value
    sm.mu.Unlock()
}
```

But don't forget that the `type` keyword is only allowed in the receiver type. For explanation, see [FAQ](#FAQ).

### Generic array lengths (unimplemented)

The original proposal also included generic array lengths. There is still an intention to support them, but I haven't implemented them yet, because this has been enough work so far. They'd work like this:

```go
func Reverse(a *[const n]type T) {
    for i, j := 0, n-1; i < j; i, j = i+1, j-1 {
        a[i], a[j] = a[j], a[i]
    }
}
```

And that's all! Happy hacking!

## FAQ

### Is this an officially accepted proposal?

No! Enjoy it, experiment with it, and don't complain about the syntax ;). Eh, you can, but you know, don't overdo it.

### Does this work?

Yep! There's only one limitation: it only translates a single file. And there's only one unimplemented feature: generic array lengths.

### What are the advantages of this syntax?

Most proposals propose a syntax that introduces another pair of parentheses in function declarations, like this:

```go
func Map(type T, U)(a []T, f func(T) U) []U {
    // ...
}
```

There are four main advantages of my syntax compared to the other proposals:
1. **It's clear where a type parameter gets inferred.** In my proposal concrete type of a type parameter gets inferred exactly where the `type` keyword is. With other proposals, it's not clear where it gets inferred and if it can be inferred at all.
2. **It's clear whether a type parameter must be specified manually by the caller.** In my proposal, if a type parameter is unnamed, it must be specified by the caller manually. Otherwise, it gets inferred from an argument. There is never a choice between specifying and inferring. In other proposals, it's not clear when the caller must specify the types manually and when they can be inferred, because it depends on the power of the type-checker.
3. **Fits in with built-in Go functions like `make` and `new`.** The unnamed type parameters even make it possible to give a type to the built-in `new` function. The `make` function is a little more [funky](https://faiface.github.io/funky-tour/), though. It would also require function overloading.
4. **No extra parentheses.** Better readability.

Furthermore, it introduces no new keywords.

### What are some other advantages of this proposal?

This is to be argued, but here's what I think:
1. **Easy to understand and read.** Doesn't introduce complexity.
2. **Orthogonal to other Go features.** For example, these generics don't collide with interfaces. Every problem is either suitable for generics, or for interfaces, (or not none of them), but rarely suitable for both.
3. **Fast and straightforward type-checking.** The type-checking is super simple. Just substitute a concrete type for a type parameter where it says `type` in the signature and you're good to go.

### Why no `type` keyword in function results?

Because it would make type-checking ambiguous. Let's say that `Read` can be written like this:

```go
func Read() type T {
    var x T
    fmt.Scan(&x)
    return x
}
```

Now, what types should it read here?

```go
name := Read()
age := Read()
fmt.Printf("%s is %d years old.", name, age)
```

Should it infer based on the `%s` and `%d` placeholders in `fmt.Printf`? I don't think so.

Forbidding the use of `type` in results forces the programmer to use unnamed type parameters when needed. They, in turn, make it possible to specify the type parameters manually for the caller.

### Why is the `type` keyword only allowed in the receiver in methods?

There two reasons for this.
1. **Methods are used to satisfy interfaces.** Methods that are generic beyond their receiver would seriously complicate this prospect. Either the interfaces would have to require generic methods, or the type-checker would have to be able to specialize generic methods for the purpose of satisfying interfaces. Both would complicate the system.
2. **Reflection.** Reflection makes it possible to discover all methods of a type at runtime. If methods generic beyond their receiver would be possible, reflection would need to be able to discover generic methods. This could be possible but would be quite complex. This is the same reason that generic functions aren't usable as values.

### Why no ability to create my own restrictions?

Because that's where all the unwanted complexity comes from.

Just take a look at Haskell. [Type clasess](https://en.wikipedia.org/wiki/Type_class) in Haskell are a way to specify your own restrictions. They are even simpler than the contracts proposed by the Go team. Yet, you get `Functor`, `Applicative`, `Monad`, `Monoid`, `Traversal`, and a whole bunch of abstract functions that don't make any sense unless you've spent two years studying them. And that's not all. There's a whole culture that makes you spend more time implementing various type classes than implementing the actually useful code.

Of course, I'm exaggerating, but just a little bit. Haskell is a great language, but complex. Also, Go would not become Haskell. But there would be the tools and people would misuse them somehow.

Furthermore, most situations for these custom restrictions are already covered by interfaces. With generics, interfaces become even stronger.

### How did you do this?

I copied the whole tree of [`"go/*"`](https://golang.org/pkg/go/) packages from the standard library. They implement parsing, importing, and type-checking of Go code. Then I extended them (namely `"go/ast"`, `"go/parser"`, `"go/printer"`, `"go/types"`) with support for generics. Parsing generics, type-checking generics. I also made them emit special information about generic calls and type instances that made it easier to implement the translating tool.

Now, the translation itself it a bit hacky. It works in passes. A single pass works like this:
1. Parse and type-check the code.
2. Find all non-generic functions.
3. In them, find all generic calls and generic type instances (their type parameters must be concrete).
4. Instantiate the found generic functions and types with the used parameters. This means copy-pasting their original generic implementation and replacing all uses of the type parameters with concrete types.
5. Replace all generic calls and type instances in the non-generic function with calls to the new, instantiated functions and types.
6. Write the result.

And I repeat this process until nothing changes. In the end, I remove all generic functions from the source and write the final result.

### Can I break it?

Sure. There are bugs, 100%. I've already caught and fixed many of them, but if you find some new, please file an issue.

### Why is the generated code so ugly?

Sorry. Blame `"go/printer"`.

### Why no tests?

This is the test.

## License

[MIT](LICENSE)

# memory, references and mutation

We have some goals that are in tension:

* No undefined behaviour
* [Mutable value semantics](https://research.google/pubs/mutable-value-semantics/)
* Equality is [egal](https://dl.acm.org/doi/pdf/10.1145/165593.165596)
* Values are data
* Reasonable performance

'No undefined behaviour' is straightforwad. It's only relevant here because it rules out manual memory management.

'Mutable value semantics' means that we allow mutation, but we don't allow observable aliasing between values ie the following program always prints two identical values:

```zest
y = x
print(y)
f(x@) // this can only change x, not y
print(y)
```

'Equality is egal' requires that values are equal if and only if they cannot be distinguished by a program ie:

```zest
if a == b {
  assert(f(a) == f(b))
}
```

'Values are data' means that any value can be serialized and deserialized to an equal value:

```zest
assert(x == deserialize(serialize(x)))
```

'Reasonable performance' is a little harder to be precise about. It includes both the obvious meaning - that idiomatic programs should perform well. But also that it should be possible to reason *about* performance using as much as possible only local information. Some properties that help with both of these meanings include:

* Store fields at known memory offsets, instead of in a hashtable.
* Default to static dispatch, instead of making every function overridable or even mutable.
* Never implicitly copy large values.
* Allow storing values inline in their container (eg list of structs) or on the stack, rather than requiring separate heap allocations for each.
* Allow passing values by reference, including references to values stored inline or on the stack.

Let's look at some options for memory management and see how these goals come into conflict.

## reference counting

The [original paper](https://research.google/pubs/mutable-value-semantics/) on mutable value semantics proposed reference-counting all values. If you want to mutate a value, the program first checks if the reference count is more than 1 and if so makes a new copy of the value.

This allows for very simple semantics, but violates reasonable performance. Does the following program copy `x`?

```zest
x[i]@ == 1
```

The answer depends on whether the value assigned to `x` is also currently assigned to any other variable anywhere in the program, which is not always possible to reason about locally.

The overhead of the reference-counting itself can be high, although it's hard to get good measurements. [Koka](https://pp.ipd.kit.edu/uploads/publikationen/ullrich19counting.pdf) and [lean](https://dl.acm.org/doi/pdf/10.1145/3453483.3454032) show that the number of refcount operations can be reduced by doing whole-program escape analysis (although they don't measure this separately from their allocation reuse, so it's hard to know the impact). [Ungar et al](https://dl.acm.org/doi/pdf/10.1145/3170472.3133843) show that in swift the savings just from switching from atomic to non-atomic operations can be as much as 2x in microbenchmarks, and [Choi et al](https://iacoma.cs.uiuc.edu/iacoma-papers/pact18.pdf) put the average refcounting overhead at 32% across some larger benchmarks. Both of these results imply that the number of refcount operations must be high [even after optimization](https://github.com/swiftlang/swift/blob/main/docs/OptimizerDesign.md), so we can't assume that lean/koka -style optimization would save us.

It's also hard to reconcile refcounting with interior pointers, since interior values don't have their own refcounts. We could switch to double pointers (pointer to refcount and pointer to value) but this increases memory usage and creates the possibility of accidentally leaking containers. We could have two different types of pointers, but this complicates the language semantics and maybe makes it hard to compile functions which are generic in whether their arguments are refcounted or interior pointers.

## borrow checking

We could instead borrow (hehe) ideas from rust, but there are incompatibilities with the rest of the zest language.

* Zest has a dynamically typed dialect that is used for staged evaluation. Hw do we borrow-check a program that we don't yet know the types for?
* 'Values are data' and 'equality is egal' require that serializing and deserializing a value produces a new but observationally indistinguishable value. In rust terms, if we serialize and deserialize a `&T` what do we get? It can't be a `T` because that would let us observe a difference with `type-of`. But if it's another `&T`, where is it borrowed from, what is it's lifetime, and when is it freed?
* Zest's type-system is a simple abstract interpretation of runtime type tags. The inferred type of an expression is the type tag of the value produced by that expression. There are no abstract types. So where would we put the lifetimes? If we store lifetimes in the type tag then lifetimes are values, which means they can be copied to any other point in the program by serializating and deserializing.
* Rust relies on the `Copy` trait to avoid having write `.clone()` all the time when using small types like integers. But zest doesn't have nominal types and so can't have an adhoc global registry of copyable types.

Zest is not the only language struggling with this though. We can take some inspiration from [C# refs](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/ref) and [oxcaml modes](https://oxcaml.org/documentation/modes/intro/). Both of these are less expressive than the rust system, but allow attaching lifetime information to variables/expressions rather than to types.

## weaving between the constraints

If a borrowed value is serialized and deserialized, we must get an owned value, otherwise who is responsible for dropping the new value? This implies that borrowed and owned values must have the same type and representation.

Borrowing a large value must produce a pointer to the original value (no implicit copies of large values). So all large owned values must also be represented as pointers. To simplify, let's define 'large' as 'larger than a pointer'. So bools/integers/floats are passed by value even when borrowed, but most values are passed by reference.

For every variable/expression the compiler will track the set of variables that might have been borrowed from (but not where those borrows might occur in the value, so we lose information relative to rust). Function parameters are assumed to be borrowed from the caller unless annotated as owned. All lvalues default to borrows, unless annotated as moves.

We can then coarsely borrow-check function bodies without knowing the types of the parameters or of other functions. In dynamically-typed code we only have to insert runtime checks at each function call to make sure that parameters annotated as owned are matched to argument expressions whose borrow-set is empty. In statically-typed code we know which function is being called so we can eliminate those checks. This is essentially the same analysis that we already implement for mutable references, so definitely tractable.

The usual rust restrictions will apply. Values can't be moved or mutated while borrows exist. A value may have either one mutable borrow or many immutable borrows.

Partial moves seem tricky. In `x[i]/move`, maybe `x` is a struct and we can use drop flags like rust. But maybe `x` is a list or map. Do we leave tombstones in the struct/list/map? Or should the move be implementated as a list `swap_remove` or a map `delete`? We can just ban partial moves for now.

This design doesn't currently allow for returning borrows from functions, but we could extend the annotations to allow C# -style ref returns. The returned value would have to be assumed to borrow from all the arguments, because when we're borrow-checking we don't yet know which function we're calling. For now though let's see how far we can get without returning borrows.

So far we have universal layout and no interior pointers. We can fix that by adding a type annotation. `embedded(T)` behaves exactly like `T` except that it is stored inline in it's container / on the stack. When we try to immutably borrow a `embedded(T)` we get a `T` - a pointer to the inline value. If we want to return a `embedded(T)` or assign to a mutable `embedded(T)` then we have to explicitly dereference-and-copy the value - `return x/embed`. This prevents accidental copies of large inline values, as is common in go, and discourages accidentally returning large inline values through many layers of generic function calls, as is common in go and rust.

## examples

In these examples, `borrow error` is detected at compile-time in all code, whereas `type error` is detected at runtime in the dynamic dialect and at compile-time in the static dialect.

Compared to rust, we have equivalents of `&T` and `&mut T` but we never allow them to appear inside another type. This allows borrow-checking to be separated from type-checking, but makes borrows effectively second-class.

```zest
// a has type i64 and is owned
a = 1 

// b has type i64 and is borrowed from a
b = a 

// borrow error: b is borrowed from a and must not outlive a
return b

// c has type i64 and is owned
// addition takes borrowed arguments and returns an owned value
c = b + 1

// d has type i64 and is borrowed from a and c
d = if cond() a else c

// borrow error: can't use borrowed values inside an owned struct
// this prevents us from having to keep track of which parts of a value are borrowed vs owned
// either a value is wholly owned (and must be freed at the end of the scope), or wholly borrowed
e = [a, c]

// this is fine
e = [a/copy, c/copy]

// borrow error: can't mutably borrow b because b already borrows from a
// (we could relax this somewhat - allow assignment but not interior mutation or creating mutable reference)
b/mut += 1

// this is fine
f = b/copy
f/mut += 1

// g borrows mutably from f
{
  g = f/mut
}

// borrow error: can't mutably borrow f while it is borrowed by g
{
  g = f/mut
  g2 = f/mut
}

// borrow error: can't mutably borrow f while it is borrowed by f/mut
foo0(f/mut, f/mut)

// borrow error: can't mutably borrow f while it is borrowed by f2
{
  f2 = f
  foo0(f/mut)
}

// i has type i64 and is borrowed from h
h = [0, 1]
[i, _] = h

// borrow error: can't move h while it is borrowed by i
[j, _] = h/move

// j has type i64 and is owned
drop(i)
[j, _] = h/move

// borrow error: h was already moved into j
[k, _] = h/move

// borrow error: b is a borrowed value and cannot be moved
[k, _] = b/move

// borrow error: the argument to move must be a single variable
// this avoids dealing with partial moves - we can have a single drop flag per owned variable
[k, _] = h.0/move

// l has type struct[i64, i64] and is heap-allocated
l = [0, 1]

// m has type embedded(struct[i64, i64]) and is stack-allocated
m = l/embed

// n has type struct[i64, i64] and is borrowed from m
n = m

// o has type struct[embedded(struct[i64, i64]), embedded(struct[i64, i64])]
// and is a single 32-byte heap allocation
o = [n/embed, n/embed]

// type error: can't embed a borrowed struct which contains non-embedded values
// this guarantees that /embed is only ever a shallow copy
p = [[0, 1], [0, 1]]
q = p/embed

// we have to explicitly copy the nested structs
q = p/copy/embed

// by default, functions expect their arguments to be borrowed
foo1 = (x) { ... }
foo1(e)

// it's fine to pass an owned value to a function that expects a borrowed value
foo1([0, 1])

// but not vice-versa
// type error: expected x to be an owned value but received a borrowed value
foo2 = (x/move) { ... }
foo2(e)

// we need to explicitly provide an owned value using move or copy
foo2(e/copy)
foo2(h/move)
```

We're going to find ourselves typing `/mut`, `/copy`, and `/move` a lot - maybe they deserve to be a single postfix character each?

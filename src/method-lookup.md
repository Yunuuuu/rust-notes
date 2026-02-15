# Method lookup

The [dot operator] will perform a lot of magic to convert types.
It will perform auto-referencing, auto-dereferencing, and coercion until types
match. The detailed mechanics of method lookup are defined [here][method
lookup], but here is a brief overview that outlines the main steps.

Suppose we have a function `foo` that has a receiver (a `self`, `&self` or `&mut
self` parameter). If we call `value.foo()`, the compiler needs to determine what
type `Self` is before it can call the correct implementation of the function.
For this example, we will say that `value` has type `T`.

We will use `fully-qualified syntax` to be more clear about exactly which type
we are calling a function on.

- First, the compiler checks if it can call `T::foo(value)` directly. This is
  called a "by value" method call.
- If it can't call this function (for example, if the function has the wrong
  type or a trait isn't implemented for `Self`), then the compiler tries to add
  in an automatic reference. This means that the compiler tries
  `<&T>::foo(value)` and `<&mut T>::foo(value)`. This is called an "autoref"
  method call.
- If none of these candidates worked, it dereferences `T` and tries again. This
  uses the `Deref` trait - if `T: Deref<Target = U>` then it tries again with
  type `U` instead of `T`. If it can't dereference `T`, it can also try
  _unsizing_ `T`. This just means that if `T` has a size parameter known at
  compile time, it "forgets" it for the purpose of resolving methods. For
  instance, this unsizing step can convert `[i32; 2]` into `[i32]` by
  "forgetting" the size of the array.

[Method lookup] is divided into two major phases:

  1. Probing ([`probe.rs`][probe]). The probe phase is when we decide what
     method to call and how to adjust the receiver.
  2. Confirmation ([`confirm.rs`][confirm]). The confirmation phase "applies"
     this selection, updating the side-tables, unifying type variables, and
     otherwise doing side-effectful things.

One way to think of method lookup is that we convert an expression of
the form `receiver.method(...)` into a more explicit [fully-qualified syntax]:

- `Trait::method(ADJ(receiver), ...)` for a trait call
- `ReceiverType::method(ADJ(receiver), ...)` for an inherent method call

Here `ADJ` is some kind of adjustment, which is typically a series of
autoderefs and then possibly an autoref (e.g., `&**receiver`). However
we sometimes do other adjustments and coercions along the way, in
particular unsizing (e.g., converting from `[T; n]` to `[T]`).

## The Probe phase

### Steps

The first thing that the probe phase does is to create a series of
*steps*. This is done by progressively dereferencing the receiver type
until it cannot be deref'd anymore, as well as applying an optional
"unsize" step. So if the receiver has type `Rc<Box<[T; 3]>>`, this
might yield:

1. `Rc<Box<[T; 3]>>`
2. `Box<[T; 3]>`
3. `[T; 3]`
4. `[T]`

### Candidate assembly

We then search along those steps to create a list of *candidates*. A
`Candidate` is a method item that might plausibly be the method being
invoked. For each candidate, we'll derive a "transformed self type"
that takes into account explicit self.

Candidates are grouped into two kinds, inherent and extension.

**Inherent candidates** are those that are derived from the
type of the receiver itself.  So, if you have a receiver of some
nominal type `Foo` (e.g., a struct), any methods defined within an
impl like `impl Foo` are inherent methods.  Nothing needs to be
imported to use an inherent method, they are associated with the type
itself (note that inherent impls can only be defined in the same
crate as the type itself).

<!--
FIXME: Inherent candidates are not always derived from impls.  If you
have a trait object, such as a value of type `Box<ToString>`, then the
trait methods (`to_string()`, in this case) are inherently associated
with it. Another case is type parameters, in which case the methods of
their bounds are inherent. However, this part of the rules is subject
to change: when DST's "impl Trait for Trait" is complete, trait object
dispatch could be subsumed into trait matching, and the type parameter
behavior should be reconsidered in light of where clauses.

Is this still accurate?
-->

**Extension candidates** are derived from imported traits.  If I have
the trait `ToString` imported, and I call `to_string()` as a method,
then we will list the `to_string()` definition in each impl of
`ToString` as a candidate. These kinds of method calls are called
"extension methods".

So, let's continue our example. Imagine that we were calling a method
`foo` with the receiver `Rc<Box<[T; 3]>>` and there is a trait `Foo`
that defines it with `&self` for the type `Rc<U>` as well as a method
on the type `Box` that defines `foo` but with `&mut self`. Then we
might have two candidates:

- `&Rc<U>` as an extension candidate
- `&mut Box<U>` as an inherent candidate

### Candidate search

Finally, to actually pick the method, we will search down the steps, trying to
match the **receiver type** against the **candidate types**. At each step, we
also consider an **auto-ref** and **auto-mut-ref** to see whether that makes any
of the candidates match. 

For each receiver `T`, we'll add `&T` and `&mut T` to the list immediately after
`T` (Please see [here][method-call-expr]).

For instance, if the receiver has type `Box<[i32;2]>`, then the receiver types
searched will be `Box<[i32;2]>`, `&Box<[i32;2]>`, `&mut Box<[i32;2]>`, `[i32;
2]` (by dereferencing), `&[i32; 2]`, `&mut [i32; 2]`, `[i32]` (by unsized
coercion), `&[i32]`, and finally `&mut [i32]`.

For each resulting receiver type, we
consider inherent candidates before extension candidates. If there are multiple
matching candidates in a group, we report an error, except that multiple impls
of the same trait are treated as a single match. Otherwise we pick the first
match we find.

In the case of our example, the first step is `Rc<Box<[T; 3]>>`,
which does not itself match any candidate. But when we autoref it, we
get the type `&Rc<Box<[T; 3]>>` which matches `&Rc<U>`. We would then
recursively consider all where-clauses that appear on the impl: if
those match (or we cannot rule out that they do), then this is the
method we would pick. Otherwise, we would continue down the series of
steps.

[dot operator]: https://doc.rust-lang.org/nomicon/dot-operator.html
[method lookup]: https://rustc-dev-guide.rust-lang.org/hir-typeck/method-lookup.html#method-lookup
[probe]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir_typeck/method/probe/
[confirm]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir_typeck/method/confirm/
[method-call-expr]: https://doc.rust-lang.org/reference/expressions/method-call-expr.html
[fully-qualified syntax]: https://doc.rust-lang.org/nightly/book/ch20-02-advanced-traits.html#fully-qualified-syntax-for-disambiguation-calling-methods-with-the-same-name

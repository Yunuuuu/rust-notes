# Safe APIs Must Not Turn Caller Logic Errors into Undefined Behavior

**A caller using only safe interfaces must not be able to trigger undefined behavior merely by
making a logical mistake. If such a use is possible, the abstraction exposing those interfaces is
unsound.** This includes mistakes in arguments, call order, callbacks, and implementations of safe
traits.

This is a requirement on the abstraction, not a claim that every existing library is bug-free. The
Rust Reference makes the author of unsafe code responsible for preventing safe clients from
triggering undefined behavior (UB). A safe counterexample demonstrates a soundness bug. [\[1\]][1]

## A documented requirement is not automatically a safety obligation

Safe APIs can impose logical requirements. Violating them may produce incorrect results, panics,
or other unspecified behavior, but must not cause UB. The Reference explicitly makes this
distinction for collection keys and the relationship between `Hash` and `Eq`. [\[2\]][2]

Consequently, "the caller violated the documentation" does not by itself excuse UB. The question
is whether the interface assigns an additional safety obligation through an unsafe contract, or
merely states a logical requirement of a safe interface.

A panic is not UB. An implementation may reject an invalid request or panic while still preserving
memory safety. It must not perform an invalid memory operation because it expected the caller to
check first.

## A broken framer exposes the distinction

Consider an ordinary trait whose documented result must be within the input:

```rust
pub trait Framer {
    /// Returns a frame end no greater than `input.len()`.
    fn frame_end(&mut self, input: &[u8]) -> usize;
}
```

A caller can violate that requirement entirely in safe Rust:

```rust
# pub trait Framer {
#     fn frame_end(&mut self, input: &[u8]) -> usize;
# }
struct BrokenFramer;

impl Framer for BrokenFramer {
    fn frame_end(&mut self, _input: &[u8]) -> usize {
        usize::MAX
    }
}
```

The implementation is logically wrong. Returning that integer is not itself UB. Now suppose the
library exposes this **unsound counterexample**:

```rust,no_run
# pub trait Framer {
#     fn frame_end(&mut self, input: &[u8]) -> usize;
# }
pub fn frame<'a, F: Framer>(framer: &mut F, input: &'a [u8]) -> &'a [u8] {
    let end = framer.frame_end(input);
    unsafe { input.get_unchecked(..end) }
}
```

Passing `BrokenFramer` and an ordinary slice lets a safe caller violate the precondition of
`get_unchecked`. Saying that `Framer` required a valid boundary does not repair the abstraction.
The framer has a logic bug; the library has a soundness bug because it turns that result into UB.
This example applies the Reference's soundness criterion to a generic framing API. [\[1\]][1]

A checked access handles the same mistake without an unchecked out-of-bounds operation:

```rust
# pub trait Framer {
#     fn frame_end(&mut self, input: &[u8]) -> usize;
# }
pub fn frame<'a, F: Framer>(framer: &mut F, input: &'a [u8]) -> &'a [u8] {
    let end = framer.frame_end(input);
    &input[..end]
}
```

An invalid end now causes a panic. Returning an error would be another API choice. Neither requires
pretending that every implementation of the safe trait is correct.

## BTreeMap uses Ord without making safety depend on its correctness

`BTreeMap` needs comparisons to locate keys. A user can nevertheless implement `Ord` using only
safe Rust and return inconsistent answers. The map may then behave incorrectly, but its unsafe
internals must remain safe. This is the example used in the Rustonomicon. [\[3\]][3]

"Do not trust `Ord`" therefore does not mean "do not call `cmp`" or "verify every ordering law."
It means that correctness of an arbitrary user's comparisons cannot be a necessary premise of
the map's memory-safety argument.

The `BTreeMap` documentation also specifies that changing a key's relative ordering while it is
stored is a logic error whose consequences remain contained in the affected map and do not include
UB. [\[4\]][4]

## Read can return a wrong length without overflowing the buffer

The discussion [Can read() overflow a buffer?][5] separates two failures that are easily confused.
A `Read` implementation can receive a buffer of length 100, write a few bytes, and return
`Ok(100000)`. Returning that number requires no unsafe operation and does not mean 100000 bytes
were actually written.

If generic consuming code uses the returned number in an unchecked operation and causes UB, that
consumer's unsafe code is at fault. If the reader itself writes beyond the buffer using incorrect
unsafe code, the fault is in that implementation or its dependencies. Replies 2, 3, and 7 explain
this distinction explicitly. [\[5\]][5]

**Responsibility for a wrong result and responsibility for memory safety are separate questions.**

This argument concerns arbitrary `Read` implementations and their returned lengths. It does not
establish that a particular reader may receive uninitialized memory; that requires a separate
argument about the initialization requirements of the operations involved.

## Trusting a specific implementation is a different claim

The Rustonomicon allows unsafe code to rely on selected, specific safe implementations, including
code in a dependency. Its example is `BTreeMap` relying on integers and slices. These are known
implementations rather than behavior supplied by an arbitrary client. [\[3\]][3]

For example, a valid slice's standard iterator and `enumerate()` provide positions for its actual
elements. Relying on those implementations is different from assuming that every user-provided
`Framer` reports a valid position merely because its trait documentation says so.

A proof about one concrete framer does not cover an open-ended `F: Framer`. If the public API accepts
arbitrary implementations, its safety argument must cover them. If it selects and encapsulates a
particular implementation, that implementation can be part of the argument. This does not permit
external safe callers to invalidate the assumptions needed for safety.

## Make additional safety obligations explicit

If an API needs to rely on the reported boundary without checking it, an unsafe trait can assign
that guarantee to implementors. Here is an illustrative contract:

```rust
/// A framer whose reported boundary can be relied on for memory safety.
///
/// # Safety
/// Every normally returning call to `frame_end` must return a value no greater
/// than the length of the input passed to that call.
pub unsafe trait TrustedFramer {
    fn frame_end(&mut self, input: &[u8]) -> usize;
}
```

Implementing it requires `unsafe impl`. The implementor now assumes the documented safety
obligation. Consumers must still use the guarantee correctly; it does not justify assumptions
absent from the contract.

Alternatively, an `unsafe fn` can put a documented obligation on its caller. A public safe API
must instead establish safety through checks, types, or encapsulation. This does not require a
runtime check on every call, but it does require an actual guarantee. The discussion
[How important is unsafe in function definition?][6], especially replies 3–5, explains this
allocation of responsibility.

## A practical review question

**Can an external caller, using only safe Rust, make this interface's unsafe internals violate
their safety conditions through a bad argument, call sequence, callback, or safe trait implementation?**

If so, the abstraction needs fixing. A documented logical requirement does not excuse the UB.
The caller remains responsible for incorrect application logic, while the safe abstraction remains
responsible for containing those mistakes within behavior that does not violate Rust's safety rules.

## References and further discussion

1. Rust project contributors. “Behavior considered undefined.” _The Rust Reference_. See the
   definition of soundness. [Reference][1].
2. Rust project contributors. “Behavior not considered unsafe,” section “Logic errors.”
   _The Rust Reference_. [Reference][2].
3. Rust project contributors. “How Safe and Unsafe Interact.” _The Rustonomicon_. Covers `BTreeMap`,
   arbitrary `Ord` implementations, specific dependencies, and unsafe traits. [Chapter][3].
4. Rust project contributors. “Struct BTreeMap.” _Rust Standard Library_. See the contract for
   changes to key ordering. [Documentation][4].
5. “Can read() overflow a buffer?” _Rust Users Forum_, May 24, 2023. Replies 2, 3, and 7 distinguish
   a wrong returned length from an actual buffer overflow. [Discussion][5].
6. “How important is unsafe in function definition?” _Rust Users Forum_, August 19, 2023.
   Replies 3–5 discuss the responsibility assumed by a safe wrapper. [Discussion][6].
7. “Using std::collections and unsafe: anything can happen.” _Rust Internals_, May 17, 2022.
   Discusses the scope and containment of unspecified behavior after logic errors. [Discussion][7].
8. “Not unsafe operation in unsafe block.” _Rust Users Forum_, May 13, 2024. Read replies 7, 14,
   and 16 together: they distinguish the root cause of UB, sound encapsulation, and reliance on
   specific safe implementations. Individual replies include disagreements and corrections.
   [Discussion][8].

[1]: https://doc.rust-lang.org/reference/behavior-considered-undefined.html
[2]: https://doc.rust-lang.org/reference/behavior-not-considered-unsafe.html#logic-errors
[3]: https://doc.rust-lang.org/nomicon/safe-unsafe-meaning.html
[4]: https://doc.rust-lang.org/std/collections/struct.BTreeMap.html
[5]: https://users.rust-lang.org/t/can-read-overflow-a-buffer/94347
[6]: https://users.rust-lang.org/t/how-important-is-unsafe-in-function-definition/98635
[7]: https://internals.rust-lang.org/t/using-std-collections-and-unsafe-anything-can-happen/16640
[8]: https://users.rust-lang.org/t/not-unsafe-operation-in-unsafe-block/111302

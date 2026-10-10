# `&mut T` Allows Mutation of Its Referent

**In `&mut T`, `mut` allows changing the referent: the `T` value that the reference points to.**
Dereferencing the reference accesses that value. [\[1\]][1]

```rust
let mut value = 1;
let reference: &mut i32 = &mut value;

*reference = 2;

assert_eq!(value, 2);
```

Here, the referent is the integer stored in `value`. The assignment changes its value from `1` to
`2`.

The same idea applies when `T` is itself a reference. In `&mut &[u8]`, the referent is a value of
type `&[u8]`. Changing that value can make it refer to a different range of bytes without changing
the bytes themselves.

## Binding mutability versus reference mutability

**Changing the reference stored in a variable and changing the value it refers to are different
operations.** `let mut` makes the variable binding mutable; `&mut T` permits mutation through the
reference. [\[2\]][2]

### A mutable binding containing a shared reference

```rust
let data = [1, 2, 3];
let mut slice: &[u8] = &data;

// Replace the reference stored in `slice`.
slice = &slice[1..];

assert_eq!(slice, &[2, 3]);
assert_eq!(data, [1, 2, 3]);

// Not allowed: the shared reference does not permit modifying the underlying referent.
// slice[0] = 9;
```

Here, `mut` belongs to the **variable binding** `slice`. Its type remains `&[u8]`, a shared slice
reference. Reassigning `slice` changes the range it refers to; it does not modify the underlying
elements. The `mut` in `let mut slice` does not turn `&[u8]` into `&mut [u8]`.

### An immutable binding containing a mutable reference

```rust
let mut data = [1, 2, 3];
let slice: &mut [u8] = &mut data;

// Modify the referent through the mutable reference.
slice[0] = 9;

assert_eq!(data, [9, 2, 3]);
```

The binding `slice` is not declared `mut`, but its type is `&mut [u8]`. Its referent is the byte
slice, so it permits changing those elements. Declaring the binding `mut` would additionally allow
replacing the reference stored in `slice` or taking a mutable borrow of that binding itself. Neither
is needed for `slice[0] = 9`. [\[2\]][2]

## A mutable reference to a slice reference

A reference is itself a value, so it can also be borrowed. In particular, **`&mut &[u8]` is a
mutable reference to a value whose type is `&[u8]`**. The outer `&mut` permits changing that
reference value; the inner `&[u8]` still provides shared access to the bytes. [\[3\]][3]

```rust
let data = [1, 2, 3];
let mut slice: &[u8] = &data;

// Mutably borrow the variable that stores the shared slice reference.
let slice_mut_ref: &mut &[u8] = &mut slice;

// Dereferencing once accesses the original `slice` variable.
*slice_mut_ref = &data[1..];

// Not allowed: the inner reference still does not permit modifying bytes.
// (*slice_mut_ref)[0] = 9;

assert_eq!(slice, &[2, 3]);
assert_eq!(data, [1, 2, 3]);
```

The layers are:

```text
slice_mut_ref        &mut &[u8]    A mutable reference to `slice`
*slice_mut_ref       &[u8]         The reference stored in `slice`
**slice_mut_ref      [u8]          The underlying byte slice
```

The assignment `*slice_mut_ref = &data[1..]` replaces the shared slice reference stored in `slice`.
The referent of the outer reference has changed, while `data` still contains `[1, 2, 3]`.

The binding `slice` needs `mut` because `&mut slice` borrows that variable mutably. The binding
`slice_mut_ref` does not need `mut`: the assignment changes its referent through `&mut`, just as
`*reference = 2` did in the first example. [\[2\]][2]

## `Read for &[u8]` changes the slice reference

The standard library implements `std::io::Read` for `&[u8]`. Reading copies bytes into the output
buffer and updates the slice reference to point to the unread remainder. The input bytes remain
unchanged. [\[4\]][4]

```rust
use std::io::Read;

let data = b"abc";
let mut reader: &[u8] = data;

let mut output = [0; 1];
reader.read_exact(&mut output).unwrap();

assert_eq!(&output, b"a");
assert_eq!(reader, b"bc");
assert_eq!(data, b"abc");
```

Both `Read::read` and `Read::read_exact` take `&mut self`. [\[4\]][4] For the implementation on
`&[u8]`, substituting the implementing type for `Self` gives:

```text
Self      = &[u8]
&mut Self = &mut &[u8]
```

The referent of this receiver is the **slice reference itself**. [\[5\]][5] The following
illustrative function uses the same idea:

```rust
fn read_prefix(reader: &mut &[u8], output: &mut [u8]) -> usize {
    let count = reader.len().min(output.len());

    output[..count].copy_from_slice(&reader[..count]);

    // Store the unread remainder back into the caller's reference.
    *reader = &reader[count..];

    count
}

let data = b"abc";
let mut reader: &[u8] = data;
let mut output = [0; 1];

assert_eq!(read_prefix(&mut reader, &mut output), 1);
assert_eq!(&output, b"a");
assert_eq!(reader, b"bc");

assert_eq!(read_prefix(&mut reader, &mut output), 1);
assert_eq!(&output, b"b");
assert_eq!(reader, b"c");
assert_eq!(data, b"abc");
```

Here, `output: &mut [u8]` permits changing the output bytes, while `reader: &mut &[u8]` permits
changing the caller's slice reference. Updating that reference preserves the read position across
calls: the first call leaves `b"bc"`, and the second leaves `b"c"`.

If the implementation were on `[u8]`, its `&mut self` would instead be `&mut [u8]`. That receiver
would provide mutable access to the bytes, without mutable access to the caller's stored slice
reference. It could not use this same technique to replace that reference with the unread remainder.

## References

1. Rust project contributors. _The Rust Reference_, “Operator expressions,” subsection “The
   dereference operator.” Online documentation, accessed October 10, 2026. [Reference][1].
2. Rust project contributors. _The Rust Reference_, “Expressions,” subsection “Mutability.” Online
   documentation, accessed October 10, 2026. [Reference][2].
3. Rust project contributors. _The Rust Reference_, “Pointer types,” section “References.” Online
   documentation, accessed October 10, 2026. [Reference][3].
4. Rust project contributors. “Trait `Read`,” including the implementation for `&[u8]`. _Rust
   Standard Library_. Online documentation, accessed October 10, 2026. [Documentation][4].
5. steffahn. Reply 2 to “Why does `[u8;1]` does not implement the read trait?” _The Rust Programming
   Language Forum_. January 18, 2022. [Reply][5].

[1]: https://doc.rust-lang.org/reference/expressions/operator-expr.html#the-dereference-operator
[2]: https://doc.rust-lang.org/reference/expressions.html#mutability
[3]: https://doc.rust-lang.org/reference/types/pointer.html
[4]: https://doc.rust-lang.org/std/io/trait.Read.html#impl-Read-for-%26%5Bu8%5D
[5]: https://users.rust-lang.org/t/why-does-u8-1-does-not-implement-the-read-trait/70519/2

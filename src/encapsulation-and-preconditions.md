# Methods Must Preserve Object State Constraints

A **precondition** is a requirement on the inputs or starting state that must hold for an
operation's promised result. For example, `binary_search` requires sorted input: the caller must
supply a sorted slice to rely on a correct search result.

**The core principle is that methods must preserve the object's state constraints.** An object may
promise to keep its data sorted or to forbid cancellation after shipping. Its methods must preserve
those rules even when callers do not check first.

A runtime check is needed when the operation could otherwise break such a rule. If the object's type
or implementation already prevents that violation, no repeated check is needed.

## No promise to keep a slice sorted: `binary_search`

**An ordinary slice does not promise to be sorted.** Unsorted data is still a valid slice.

[`binary_search`](https://doc.rust-lang.org/std/primitive.slice.html#method.binary_search) requires
the caller to supply sorted data. It does not check the ordering or promise to report an error for
unsorted input.

```rust
let values = [1, 3, 5, 7];

assert_eq!(values.binary_search(&5), Ok(2));
assert_eq!(values.binary_search(&4), Err(2));
```

Here, `Err(2)` means the value was not found and belongs at index 2; it is not an ordering error.
With unsorted input, the search result has no correctness guarantee, but the slice itself remains
valid. Sortedness is a requirement of this search operation, not an invariant of an ordinary slice.

## Protect the object's state transitions: `Order::cancel`

Suppose an `Order` promises that a shipped order cannot be cancelled. Callers may attempt
cancellation in any state, but success is not guaranteed. Calling `cancel()` on a shipped order is
allowed: the method returns `Err(CancelError::AlreadyShipped)` and leaves the state unchanged.

The method must therefore check the state before changing it:

```rust
impl Order {
    pub fn cancel(&mut self) -> Result<(), CancelError> {
        match self.status {
            OrderStatus::Pending => {
                self.status = OrderStatus::Cancelled;
                Ok(())
            }
            OrderStatus::Shipped => Err(CancelError::AlreadyShipped),
            OrderStatus::Cancelled => Ok(()),
        }
    }
}
```

Blindly assigning `Cancelled` would violate the promise for a shipped order. Returning
`AlreadyShipped` without changing the state preserves it. This protection must work even when the
caller does not first call `can_cancel()`.

The same distinction appears in
[An `if` can merely react to a domain decision](./object-coordination.md#1-an-if-can-merely-react-to-a-domain-decision).
The caller's `if` reacts to the ATM's `can_dispense_money` decision; `dispense_money` must still
reject withdrawals that would violate the ATM's state constraints, even if the caller skips that
query.

## A separate concern: method argument requirements

Protecting object state is not the only reason to check an argument.
[`copy_from_slice`](https://doc.rust-lang.org/std/primitive.slice.html#method.copy_from_slice)
promises to panic if the source and destination lengths differ. Equal lengths succeed:

```rust
let source = [1, 2, 3];
let mut destination = [0; 3];

destination.copy_from_slice(&source);
assert_eq!(destination, source);
```

Unequal lengths trigger the method's check:

```rust,should_panic
let source = [1, 2, 3];
let mut destination = [0; 2];

destination.copy_from_slice(&source); // Panics: the lengths differ.
```

The panic reports:

```text
copy_from_slice: source slice length (3) does not match destination slice length (2)
```

Both slices are valid objects even when their lengths differ. Equal lengths are a requirement of
this copying method, not an invariant of either slice. The method checks them because it promises to
reject a mismatch. **Object state constraints and method argument requirements are separate reasons
for checking a condition.**

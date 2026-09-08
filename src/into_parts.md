# `into_parts()` in Rust: Defining a Boundary Between Domain Encapsulation and Ownership Transfer

When designing domain models in Rust, a common problem appears:

A type needs private fields to preserve domain invariants, but downstream code occasionally needs
ownership of those fields without cloning large values such as `String` or `Vec`.

Consider an order:

```rust
pub struct Order {
    id: String,
    customer_id: String,
    items: Vec<OrderItem>,
}
```

An `Order` may need to guarantee that:

- the order ID is not empty;
- the order contains at least one item;
- item quantities are greater than zero;
- item prices are not negative;
- the object always remains in a valid state.

Making every field public weakens those guarantees:

```rust
pub struct Order {
    pub id: String,
    pub customer_id: String,
    pub items: Vec<OrderItem>,
}
```

A caller could write:

```rust
order.items.clear();
```

The `Order` object would still exist, but it might no longer satisfy the invariant that an order
must contain at least one item.

A common alternative is to keep `Order` opaque and provide a consuming decomposition method:

```rust
pub struct Order {
    id: String,
    customer_id: String,
    items: Vec<OrderItem>,
}

pub struct OrderParts {
    pub id: String,
    pub customer_id: String,
    pub items: Vec<OrderItem>,
}

impl Order {
    pub fn into_parts(self) -> OrderParts {
        OrderParts {
            id: self.id,
            customer_id: self.customer_id,
            items: self.items,
        }
    }
}
```

The caller can then write:

```rust
let parts = order.into_parts();

save_order_id(parts.id);
process_customer(parts.customer_id);
ship_items(parts.items);
```

No deep copy occurs here.

Ownership of the `String` and `Vec<OrderItem>` values moves from `Order` into `OrderParts`. Their
heap allocations are not duplicated.

## `into_parts()` Is Not Field Access

An `into_parts()` method is not merely another form of getter.

A normal getter says:

> I am keeping this domain object alive and temporarily observing its data.

For example:

```rust
impl Order {
    pub fn id(&self) -> &str {
        &self.id
    }

    pub fn items(&self) -> &[OrderItem] {
        &self.items
    }
}
```

After calling these methods, the `Order` still exists:

```rust
let id = order.id();
let items = order.items();

println!("{id}: {}", items.len());
```

By contrast, `into_parts()` says:

> I no longer need this object as a complete `Order`. I am ending its lifetime as a domain object
> and taking ownership of its internal data.

```rust
let parts = order.into_parts();

// `order` has been consumed and cannot be used again.
```

This creates a clear one-way boundary:

```text
Order ──into_parts()──> OrderParts
```

Before decomposition, `Order` is responsible for preserving its invariants.

After decomposition, there is no longer an `Order` instance. The caller may freely manipulate the
fields in `OrderParts` without corrupting a still-living valid order.

## Why Not Just Clone?

Suppose another component needs ownership of the order item list:

```rust
pub struct Shipment {
    items: Vec<OrderItem>,
}
```

If the function only receives `&Order`, it may need to copy the entire collection:

```rust
fn create_shipment(order: &Order) -> Shipment {
    Shipment {
        items: order.items().to_vec(),
    }
}
```

The call to `to_vec()` duplicates every `OrderItem`.

If the original order is no longer needed after the shipment is created, the function can consume it
instead:

```rust
fn create_shipment(order: Order) -> Shipment {
    let parts = order.into_parts();

    Shipment {
        items: parts.items,
    }
}
```

The item vector is moved into `Shipment`; its underlying allocation is not copied.

A useful decision order is:

```text
Only read the data
    -> borrow `&Order`

Transfer ownership of internal fields
and no longer use the original `Order`
    -> call `into_parts(self)`

Keep both the original `Order`
and an independent copy
    -> clone
```

Private fields do not imply that cloning is necessary.

Cloning is appropriate only when two independently owned copies are genuinely required.

## Why Not Make Every `Order` Field Public?

Public fields make ownership extraction convenient:

```rust
let Order {
    id,
    customer_id,
    items,
} = order;
```

But they also allow unrestricted mutation while the object is still supposed to represent a valid
order:

```rust
order.items.clear();
order.id.clear();
```

That may be perfectly acceptable when `Order` is only a passive data structure.

It is less appropriate when `Order` means “an order that is always valid.”

`into_parts()` establishes a more explicit boundary:

```text
While `Order` exists
    -> the type preserves its invariants

After `Order` is consumed
    -> the caller owns the underlying data
```

The method does not prevent callers from obtaining the data. It restricts unrestricted decomposition
to the point where the domain object stops existing.

## `OrderParts` Is Still Public API

The following type is unquestionably part of the public API:

```rust
pub struct OrderParts {
    pub id: String,
    pub customer_id: String,
    pub items: Vec<OrderItem>,
}
```

Publishing `OrderParts` commits the library to:

- the field names;
- the field types;
- the field visibility;
- the overall decomposition shape.

The advantage of `into_parts()` is therefore not that it avoids exposing a public data structure.

Its real advantage is this:

> The internal representation of `Order` does not have to be identical to the public decomposition
> representation.

The current implementation might be:

```rust
pub struct Order {
    id: String,
    customer_id: String,
    items: Vec<OrderItem>,
}
```

A future implementation could be:

```rust
pub struct Order {
    identity: OrderIdentity,
    contents: Box<OrderContents>,
}
```

As long as the library can still implement:

```rust
impl Order {
    pub fn into_parts(self) -> OrderParts {
        // Convert the internal representation into the stable public
        // decomposition representation.
    }
}
```

callers do not need to know how `Order` stores its data internally.

## Comparison With an Internal Public Data Object

Another possible design is to define a public data object and store it directly inside the domain
object:

```rust
#[non_exhaustive]
pub struct OrderData {
    pub id: String,
    pub customer_id: String,
    pub items: Vec<OrderItem>,
}

pub struct Order {
    data: OrderData,
}

impl Order {
    pub fn data(&self) -> &OrderData {
        &self.data
    }

    pub fn into_data(self) -> OrderData {
        self.data
    }
}
```

This design also supports zero-copy ownership transfer:

```rust
let data = order.into_data();
let items = data.items;
```

However, it makes a stronger architectural commitment:

> `OrderData` is the canonical data representation of `Order`, and callers may borrow that complete
> representation.

Once the API exposes:

```rust
pub fn data(&self) -> &OrderData
```

the internal design of `Order` usually needs to preserve a stable `OrderData` value that can be
borrowed.

By comparison:

```rust
pub fn into_parts(self) -> OrderParts
```

only promises that the object can produce a particular decomposition after being consumed. It does
not require the object to store its data internally in exactly that form.

The distinction can be summarized as follows:

```text
into_parts()
    exposes the decomposition representation after consumption
    does not require internal storage to match that representation

data() -> &OrderData
    exposes a borrowable canonical representation
    usually binds the internal structure to that representation
```

When `OrderData` is intentionally the long-term canonical representation, storing it directly is
reasonable.

When the internal representation should remain independently evolvable, `into_parts()` provides more
flexibility.

## What Is `OrderParts` in DDD?

`OrderParts` usually does not correspond to one of the classic DDD tactical patterns.

It is generally not:

- an Entity;
- an Aggregate;
- a Domain Service;
- a Repository;
- a Factory;
- an Application Service.

A more accurate description is:

> A decomposed representation of a domain object, or a domain-layer data carrier.

`Order` carries the domain semantics:

```rust
pub struct Order {
    id: String,
    customer_id: String,
    items: Vec<OrderItem>,
}
```

It may be responsible for:

- preserving order invariants;
- adding and removing items;
- calculating totals;
- deciding whether cancellation is allowed;
- enforcing state transitions.

`OrderParts` only contains the data extracted from the consumed object:

```rust
pub struct OrderParts {
    pub id: String,
    pub customer_id: String,
    pub items: Vec<OrderItem>,
}
```

Because all of its fields are public, callers can usually modify them freely. It therefore no longer
carries the full guarantee of “a valid order.”

## Is `OrderParts` a Value Object?

Usually not.

A Value Object normally has a clear domain meaning and remains valid as a self-contained value. For
example:

```rust
pub struct Money {
    amount: Decimal,
    currency: Currency,
}
```

`Money` is a complete domain concept.

`OrderParts`, by contrast, often exists only to support:

- ownership decomposition;
- zero-copy transfer;
- routing different fields to different downstream components;
- ending the lifetime of the original domain object.

There is no need to force it into the Value Object category.

It would only be reasonable to treat it as a Value Object if “order components” were themselves a
meaningful concept in the domain language and the type preserved its own rules and validity.

Otherwise, “decomposition type” or “domain data carrier” is more precise.

## Is `OrderParts` a DTO?

Not necessarily.

A DTO usually serves a cross-layer or external boundary, such as:

- an HTTP request or response;
- a database record;
- a message queue payload;
- an RPC parameter;
- a serialization format.

For example:

```rust
pub struct OrderResponseDto {
    pub id: String,
    pub customer_id: String,
    pub item_count: usize,
}
```

Its structure is normally determined by an external protocol or presentation requirement.

If `OrderParts` is defined inside the domain layer and exists only to expose the owned components of
`Order`, it is better understood as a domain-layer helper type than as a transport DTO.

The important question is not whether its fields are public. The important question is why the type
exists:

```text
Created for HTTP, persistence, or message transport
    -> DTO

Created to expose the owned components of a consumed domain object
    -> Parts type or domain data carrier
```

## When Should a Type Provide `into_parts()`?

An `into_parts()` method is useful when:

1. the type preserves meaningful invariants;
2. its fields should not be freely mutated while the object remains alive;
3. callers genuinely need ownership of multiple internal fields;
4. the original object is no longer needed afterward;
5. the fields may contain large owned buffers such as `String` or `Vec`;
6. cloning should be avoided;
7. the public decomposition representation should remain separate from the internal storage
   representation.

Typical usage looks like this:

```rust
let OrderParts {
    id,
    customer_id,
    items,
} = order.into_parts();
```

## When Should a Type Not Provide `into_parts()`?

When callers only perform read-only calculations:

```rust
fn calculate_total(order: &Order) -> Money
```

borrowing is sufficient.

When callers only need a specific domain transformation:

```rust
fn create_invoice(order: Order) -> Invoice
```

a direct transformation may communicate intent better than exposing every part first.

When the type is already an unconstrained passive data structure:

```rust
pub struct SearchResult {
    pub title: String,
    pub url: String,
    pub score: f64,
}
```

public fields are often simpler than designing an `into_parts()` API.

An `into_parts()` method should therefore not be added merely because the fields are private.

The relevant question is:

> Does the caller need to end the lifetime of this domain object and take ownership of several
> internal fields without copying them?

Only when the answer is yes does `into_parts()` have a clear purpose.

## Conclusion

`into_parts()` is an ownership-boundary design.

It allows a domain object to preserve encapsulation and invariants while it exists, while still
allowing callers to obtain ownership of its internal data when the object’s lifecycle ends.

Its main value is not reducing getter boilerplate or avoiding every public data type. Its value is
that it:

- explicitly marks the end of the object’s lifecycle;
- transfers owned fields without cloning;
- prevents callers from corrupting a still-living domain object;
- separates the public decomposition representation from internal storage;
- uses Rust’s ownership system to prevent the original object from being reused after decomposition.

The design can be understood as:

```text
Complete object protected by domain rules
                │
                │ consume
                ▼
Independently owned component data
```

When callers only need to observe the data, borrow it.

When callers need to consume the object and take ownership of its resources, use `into_parts()`.

When callers need both the original object and an independent copy, clone.

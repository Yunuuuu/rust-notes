# Context Should Follow Meaning, Not Usage

A `Context` often appears because one operation depends on information accumulated earlier.

A parser may need previous input. A traversal may need the current path. A protocol handler may need
the current phase. A workflow may need earlier decisions.

The easy mistake is to attach the context to whichever object uses it most.

```rust
struct Parser {
    context: Context,
}
```

But usage is not ownership.

The better question is:

> **What does this context describe?**

Suppose a reusable parser processes two independent sources:

```text
source A <-> context A
source B <-> context B
```

The parser uses both contexts, but neither context describes the parser.

Each context describes the history of one particular source.

That suggests a different boundary:

```rust
struct Cursor<S> {
    source: S,
    context: Context,
}
```

Now the structure reflects the real relationship:

```text
Cursor A
├── Source A
└── Context A

Cursor B
├── Source B
└── Context B
```

The parser remains reusable:

```rust
struct Parser;
```

and can operate on either pair:

```rust
parser.parse(input, &mut cursor.context)?;
```

The important point is that the context follows the continuity whose history it represents.

This applies beyond parsers.

A context may belong to:

```text
a request
a transaction
a conversation
a workflow instance
a traversal
a protocol exchange
a document
a processing session
```

An object may read or modify that context without owning it.

## Sharing information does not imply sharing ownership

Another object may need information derived from the context.

That does not mean the context itself must be exposed.

The owner can provide:

```rust
owner.current_scope()
```

or:

```rust
owner.snapshot()
```

or:

```rust
owner.view()
```

It helps to distinguish:

```text
needs a result
    -> expose behavior

needs observation
    -> expose a view or snapshot

needs independent ownership or mutation authority
    -> reconsider the boundary
```

## The useful questions

When deciding where a context belongs, ask:

1. What is this context about?
2. What object represents that continuity?
3. Who owns its lifecycle?
4. Who is allowed to mutate it?
5. Does another object need the context itself, or only information derived from it?

A useful rule of thumb is:

> **Bind context to the thing whose history, lifecycle, or continuity gives that context meaning—not
> merely to the object that happens to use it.**

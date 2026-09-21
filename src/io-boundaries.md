# Do Not Confuse Input boundaries with Domain Meaning

A structured input usually has at least two different kinds of boundaries:

- **physical boundaries**, determined by how the input is encoded or transported;
- **semantic boundaries**, determined by what the input means.

Those boundaries may happen to coincide, but they are not the same thing.

Consider a text file read from a byte stream.

The underlying reader does not naturally produce complete lines. It may return arbitrary chunks:

```text
chunk 1: "name = ali"
chunk 2: "ce\nage = "
chunk 3: "20\n"
```

Before anything can interpret the content, something has to reconstruct the physical lines:

```text
"name = alice"
"age = 20"
```

That work involves understanding the representation of the input:

```text
bytes
  ↓
detect line terminators
  ↓
preserve incomplete data between reads
  ↓
produce complete physical lines
```

I consider that infrastructure.

It is concerned with **how the data arrives**, not with **what the data means**.

A concrete implementation might therefore expose something like:

```rust
pub struct LineReader<R> {
    reader: R,
    // buffering state
}

impl<R> LineReader<R> {
    pub fn next_line(
        &mut self,
    ) -> Result<Option<Line>, ReadError> {
        // reconstruct one complete physical line
        todo!()
    }
}
```

Whether it uses `BytesMut`, an internal buffer, memory mapping, synchronous I/O, or asynchronous I/O
is an implementation detail.

The higher layers only care that a complete `Line` is produced.

## A physical line is not necessarily a domain record

Suppose the current format happens to define:

```text
one line = one record
```

It is easy to collapse the two concepts:

```text
Line == Record
```

I prefer not to do that too early.

The physical boundary answers:

> Where does this unit of encoded input end?

The semantic boundary answers:

> What domain concept does this input represent?

Those are different questions.

Today:

```text
line 1 -> record 1
line 2 -> record 2
```

Tomorrow the format may define:

```text
lines 1-4 -> record 1
```

or:

```text
lines continue until a terminator
```

or:

```text
an indented line continues the previous record
```

The infrastructure responsibility has not changed:

```text
bytes -> physical lines
```

What changed is the interpretation:

```text
physical lines -> logical record
```

That distinction keeps the I/O boundary stable while allowing the input format to evolve.

## The abstraction should describe what the caller actually needs

A higher layer may define a small input abstraction:

```rust
pub trait Lines {
    type Error;

    fn next_line(
        &mut self,
    ) -> Result<Option<Line>, Self::Error>;
}
```

A concrete infrastructure adapter implements it:

```rust
impl<R> Lines for LineReader<R> {
    type Error = ReadError;

    fn next_line(
        &mut self,
    ) -> Result<Option<Line>, Self::Error> {
        // byte reading and line framing
        todo!()
    }
}
```

The dependency direction is then:

```text
higher-level abstraction
        ▲
        │ implements
infrastructure adapter
```

The higher-level code does not need to know whether the line came from:

```text
a file
a compressed stream
memory
a network connection
```

Nor does it need to know how incomplete reads were handled.

## Domain meaning starts after framing

Once a complete physical unit exists, domain interpretation can begin.

For example:

```text
bytes
  ↓
LineReader
  ↓
Line
  ↓
Parser
  ↓
Record
```

If several physical lines are needed:

```text
bytes
  ↓
LineReader
  ↓
Line
  ↓
record assembly
  ↓
Parser
  ↓
Record
```

The important separation is:

> **Infrastructure reconstructs the representation. Domain logic interprets its meaning.**

A newline may be significant to the domain format, but scanning a byte buffer to discover that
newline is still a technical concern.

I find this distinction useful because it prevents implementation mechanics such as buffering, chunk
sizes, partial reads, and newline handling from leaking into the domain model.

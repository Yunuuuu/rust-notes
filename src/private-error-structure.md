# Rust Error APIs: Keep Internal Error Structure Private

**Keep internal error structure private wherever possible.**

A Rust library often encounters several kinds of internal errors: failed reads, missing fields,
numeric parsing failures, and operations that the object's current state does not permit. Those
internal errors do not all need to become public types, variants, or fields.

Once callers can construct or match those details, changing them can break external code. Keeping
the representation private leaves fewer implementation details in the public interface. [\[1\]][1]
Callers can still receive useful messages, inspect selected information, and handle errors through
the interfaces the library exposes.

## `std::io::Error`

`std::io::Error` is a struct with private fields. Callers use its methods without knowing how an
error is stored:

```rust
use std::io::{Error, ErrorKind};

let error = Error::new(ErrorKind::InvalidData, "port is not an integer");

assert_eq!(error.kind(), ErrorKind::InvalidData);
assert_eq!(error.to_string(), "port is not an integer");
assert!(error.get_ref().is_some());
```

The caller can inspect `kind()` and display the error while the storage remains private.
`raw_os_error()` can return an operating system error code, and `get_ref()` can return a wrapped
error, when that information is present. The public API provides access to these details without
exposing the fields that hold them. [\[2\]][2]

## `serde_json::Error`

`serde_json::Error` also has private fields. In version 1.0.151, callers can use methods such as
`is_eof()`, `line()`, and `column()` to inspect a parsing failure, and display the error directly.
[\[3\]][3]

```rust
let error = serde_json::from_str::<serde_json::Value>("{").unwrap_err();

assert!(error.is_eof());
assert_eq!(error.line(), 1);
println!("{error}");
```

Inside the crate, an `ErrorImpl` holds an `ErrorCode` and the input position. `ErrorCode` contains
cases such as `EofWhileParsingObject`, `EofWhileParsingString`, and `ExpectedColon`, but it is
visible only within the crate. External callers can inspect the failure through `Error` without
depending on that internal enum or its fields. [\[4\]][4]

## A public error around a private enum

The `thiserror` documentation shows a public wrapper around a private error representation.
`#[error(transparent)]` forwards `Display` and `source()` to the wrapped error. [\[5\]][5]

For example, a configuration loader can return one public `ConfigError` while keeping its error enum
private. This version exposes the affected field so that a caller can identify the input to correct.

The example uses `thiserror = "2.0.20"`. It accepts any value that parses as a `u16`; additional
application restrictions would need their own validation.

```rust
use std::num::ParseIntError;
use thiserror::Error;

#[derive(Debug, Error)]
#[error(transparent)]
pub struct ConfigError(ErrorRepr);

#[derive(Debug, Error)]
enum ErrorRepr {
    #[error("missing configuration field `{field}`")]
    MissingField { field: &'static str },

    #[error("invalid integer for configuration field `{field}`")]
    InvalidNumber {
        field: &'static str,
        #[source]
        source: ParseIntError,
    },
}

impl ConfigError {
    pub fn field(&self) -> &'static str {
        match &self.0 {
            ErrorRepr::MissingField { field }
            | ErrorRepr::InvalidNumber { field, .. } => field,
        }
    }
}

pub fn parse_port(value: Option<&str>) -> Result<u16, ConfigError> {
    let value = value.ok_or(ConfigError(ErrorRepr::MissingField {
        field: "port",
    }))?;

    value.parse::<u16>().map_err(|source| {
        ConfigError(ErrorRepr::InvalidNumber {
            field: "port",
            source,
        })
    })
}

let error = parse_port(Some("abc")).unwrap_err();

assert_eq!(error.field(), "port");
println!("{error}");

if let Some(source) = std::error::Error::source(&error) {
    println!("caused by: {source}");
}
```

Callers receive `ConfigError`. They can read `field()`, print the error, and inspect its source, but
code outside the defining module cannot match `ErrorRepr::InvalidNumber` or access its fields. The
library builds that private variant in `map_err`, where the field name is available.

Changing or splitting `ErrorRepr::InvalidNumber` later does not require callers to update a match on
that variant, because it was never available to them. The public methods and their promised behavior
still need to remain compatible.

This example deliberately exposes the `ParseIntError` through `source()`. Such a source can be
downcast to its concrete type, so wrapping the error does not make the underlying cause
inaccessible. [\[6\]][6] What stays private here is the surrounding `ErrorRepr` and its structure.

## A public enum with a private payload structure

A public enum can keep its payload's fields private by putting them in a separate struct. The Cargo
compatibility guide illustrates this approach for controlling the visibility of enum payload fields.
[\[1\]][1]

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum LoadError {
    #[error(transparent)]
    Invalid(InvalidConfig),
}

#[derive(Debug, Error)]
#[error("invalid configuration field `{field}`: {message}")]
pub struct InvalidConfig {
    field: &'static str,
    message: String,
}
```

Callers can match `LoadError::Invalid(details)` and display `details`. They cannot directly read or
destructure `field` and `message` from outside the defining module. Although this introduces an
additional public type name, it keeps those fields out of the public interface. Putting the fields
directly in a public enum variant would expose them.

The `LoadError::Invalid` variant itself remains public. Adding `#[non_exhaustive]` to the enum would
reserve room for future variants, but would not hide an existing variant or its payload. The private
fields in `InvalidConfig` provide the encapsulation in this example. [\[7\]][7]

## References

For living documentation, the dates below are access dates rather than publication dates.

1. Rust project contributors. _The Cargo Book: SemVer Compatibility_. Sections on public items, enum
   fields, and private fields. Accessed October 9, 2026. [Official documentation][1].
2. Rust project contributors. _Rust Standard Library: `std::io::Error`_. Methods `new`, `kind`,
   `raw_os_error`, and `get_ref`. Accessed October 9, 2026. [Official documentation][2].
3. Serde project contributors. _`serde_json::Error`_, version 1.0.151. Methods `is_eof`, `line`, and
   `column`. Accessed October 9, 2026. [Versioned API documentation][3].
4. Serde project contributors. _`serde_json` source: `src/error.rs`_, version 1.0.151. `ErrorImpl`
   and `ErrorCode`, lines 230–256. Accessed October 9, 2026. [Versioned source][4].
5. David Tolnay and contributors. _`thiserror`_, version 2.0.20. “Details”: transparent errors and
   opaque error representations. Accessed October 9, 2026. [Versioned documentation][5].
6. Rust project contributors. _Rust Standard Library: `std::error::Error`_. Methods `source` and
   `downcast_ref`. Accessed October 9, 2026. [Official documentation][6].
7. Rust project contributors. _The Rust Reference: Type System Attributes_, “The `non_exhaustive`
   attribute.” Accessed October 9, 2026. [Official documentation][7].

[1]: https://doc.rust-lang.org/cargo/reference/semver.html
[2]: https://doc.rust-lang.org/std/io/struct.Error.html
[3]: https://docs.rs/serde_json/1.0.151/serde_json/struct.Error.html
[4]: https://docs.rs/serde_json/1.0.151/src/serde_json/error.rs.html#230-256
[5]: https://docs.rs/thiserror/2.0.20/thiserror/#details
[6]: https://doc.rust-lang.org/std/error/trait.Error.html
[7]: https://doc.rust-lang.org/reference/attributes/type_system.html#the-non_exhaustive-attribute

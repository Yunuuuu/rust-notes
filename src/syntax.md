# Syntax

## Namespace

A
[namespace](https://doc.rust-lang.org/reference/names/namespaces.html#namespaces)
is a logical grouping of declared names. Names are segregated into separate
namespaces based on the kind of entity the name refers to. Namespaces allow the
occurrence of a name in one namespace to not conflict with the same name in
another namespace.

## Items

[Items](https://doc.rust-lang.org/reference/items.html) are entirely determined
at compile-time, generally remain fixed during execution, and may reside in
read-only memory.

Item names from outer modules are not in scope within a nested module. A
[path](https://doc.rust-lang.org/reference/paths.html) may be used to refer to
an item in another module.

## Patterns

[identifier
patterns](https://doc.rust-lang.org/reference/patterns.html#identifier-patterns)
bind the value they match to a variable in the [value
namespace](https://doc.rust-lang.org/reference/names/namespaces.html#r-names.namespaces.kinds).

The variable will **shadow** any variables of the same name in scope. The scope
of the new binding depends on the context of where the pattern is used (such as
a `let` binding or a `match` arm).

By default, [identifier
patterns](https://doc.rust-lang.org/reference/patterns.html#identifier-patterns)
bind a variable to a copy of or move from the matched value depending on whether
the matched value implements
[Copy](https://doc.rust-lang.org/reference/special-types-and-traits.html#copy).
This can be changed to bind to a reference by using the `ref` keyword, or to a
mutable reference using `ref mut`.

```rust,ignore
match a {
    None => (),
    Some(value) => (),
}

match a {
    None => (),
    Some(ref value) => (),
}
```

In the first match expression, the value is copied (or moved). In the second
match, a reference to the same memory location is bound to the variable value.
This syntax is needed because in destructuring subpatterns the `&` operator
can't be applied to the value's fields.

`ref` is not something that is being matched against. Its objective is
exclusively to make the matched binding a reference, instead of potentially
copying or moving what was matched.

To service better ergonomics, patterns operate in different [binding
modes](<https://doc.rust-lang.org/reference/patterns.html#binding-modes>) in
order to make it easier to bind references to values. 

* Each time a reference is matched using a non-reference pattern, it will
  automatically dereference the value and update the default binding mode.

* When a reference value is matched by a non-reference pattern, it will be
  automatically treated as a `ref` or `ref mut` binding. 

```rust
let x: &Option<i32> = &Some(3);
if let Some(y) = x {
    // x was automatically derefrenced
    // y was converted to `ref y` and its type is &i32
}
```

## Expression statements

>An [expression
statement](https://doc.rust-lang.org/reference/statements.html#expression-statements)
is one that evaluates an expression and ignores its result. As a rule, an
expression statement's purpose is to trigger the effects of evaluating its
expression.

The purpose of **expression statement** is the effect of evaluation, so using it
just to drop the result of evaluation would go against this purpose. This means
that when an **expression statement** evaluates a single variable and ignores
the result, the variable may be considered moved, and its ownership may change
after the statement is executed. See
[issue](https://users.rust-lang.org/t/the-expression-without-effect-moves-the-variable/110239)

As a general rule, the following two statements are functionally equivalent:

```rust,ignore
EXPRESSION;
```

and
```rust,ignore
let _ = EXPRESSION; // or `drop(EXPRESSION)`
```

For example:

```rust
struct PrintOnDrop(&'static str);

impl Drop for PrintOnDrop {
    fn drop(&mut self) {
        println!("{}", self.0);
    }
}

let moved;
// No destructor run on assignment.
moved = PrintOnDrop("Drops when moved");
println!("Before expression statement");
moved; // Drops now
println!("After expression statement");
```

## Expressions
[Expressions](<https://doc.rust-lang.org/reference/expressions.html#place-expressions-and-value-expressions>)
are divided into two main categories: place expressions and value expressions:

* A place expression is an expression that represents a memory location.

* A value expression is an expression that represents an actual value.

The following contexts are place expression contexts:

  - The left operand of a [compound assignment](https://doc.rust-lang.org/reference/expressions/operator-expr.html#compound-assignment-expressions) expression.
  - The operand of a unary [borrow](https://doc.rust-lang.org/reference/expressions/operator-expr.html#borrow-operators), [raw borrow](https://doc.rust-lang.org/reference/expressions/operator-expr.html#raw-borrow-operators) or [dereference](https://doc.rust-lang.org/reference/expressions/operator-expr.html#the-dereference-operator) operator.
  - The operand of a field expression.
  - The indexed operand of an array indexing expression.
  - The operand of any implicit borrow.
  - The initializer of a let statement.
  - The [scrutinee](https://doc.rust-lang.org/reference/glossary.html#scrutinee) of an if let, match, or while let expression.
  - The base of a functional update struct expression.

When a place expression is evaluated in a value expression context, or is bound
by value in a pattern, it denotes the value held in that memory location. If the
type of that value implements Copy, then the value will be copied. In the
remaining situations, if that type is Sized, then it may be possible to move the
value. After moving out of a place expression that evaluates to a local
variable, the location is deinitialized and cannot be read from again until it
is reinitialized.

## Dereference
The `*`
([dereference](<https://doc.rust-lang.org/reference/expressions/operator-expr.html#the-dereference-operator>))
operator is also a unary prefix operator. 

* When applied to a
  [pointer](https://doc.rust-lang.org/reference/types/pointer.html) it denotes
  the pointed-to location.

* If the expression is of type `&mut T` or `*mut T`, and is either a local
  variable, a (nested) field of a local variable or is a mutable place
  expression, then the resulting memory location can be assigned to.

## Reference

References come in two kinds:

 - A **shared reference** lets you read but not modify its referent. However,
you can have as many shared references to a particular value at a time as you
like.  The expression `&e` yields a shared reference to `e`’s value; if `e` has
the type `T`, then `&e` has the type `&T`, pronounced “**ref T**.” Shared
references are `Copy`.

 - If you have a **mutable reference** to a value, you may both read and modify
the value. However, you may not have any other references of any sort to that
value active at the same time. The expression `&mut e` yields a mutable
reference to `e`’s value; you write its type as `&mut T`, which is pronounced
“**ref mute T**.” Mutable references are not `Copy`.

### Fat pointer

A fat pointer, two-word values carrying the address of some value, along with
some further information necessary to put the value to use.

 - A reference to a slice is a fat pointer, carrying the starting address of the
slice and its length

 - A trait object, a reference to a value that implements a certain trait. A
trait object carries a value’s address and a pointer to the trait’s imple‐
mentation appropriate to that value, for invoking the trait’s methods.


### Reference lifetime

 - If you have a variable `x`, then a reference to `x` must not outlive `x`
   itself.

 - If you store a reference in a variable `r`, the reference's type must be good
   for the entire lifetime of the variable, from its initialization until its
   last use.


### Memory Reallocation

A mutable reference to a collection (`&mut Vec<T>`) points to the collection's
descriptor (the header). This descriptor contains the metadata—**pointer**,
**length**, and **capacity**—necessary to manage the heap. When you call
`.push()`, the collection may reallocate its internal heap buffer and update its
internal pointer to a new location. The mutable reference to the `Vec` remains
valid throughout this process because it points to the stable descriptor, not
the shifting heap data itself.

In contrast, a mutable slice (`&mut [T]`) is a fat pointer that points directly
to the heap data. Because a slice is a fixed-size 'window', it lacks the
metadata to manage capacity or request more memory. Furthermore, a slice cannot
`.push()` because resizing the underlying buffer could trigger a reallocation.
If the data moved, the slice's direct pointer would become a dangling pointer to
deallocated memory. 

## impl trait

[impl
Trait](https://doc.rust-lang.org/reference/types/impl-trait.html#impl-trait)
provides ways to specify unnamed but concrete types that implement a specific
trait. It can appear in two sorts of places: argument position (where it can act
as an anonymous type parameter to functions), and return position (where it can
act as an abstract return type). 

`impl Trait` in argument position is syntactic sugar for a generic type
parameter like `<T: Trait>`, except that the type is anonymous and doesn’t
appear in the [GenericParams](https://doc.rust-lang.org/reference/items/generics.html#generic-parameters) list.

Functions can use `impl Trait` to return an abstract return type. These types
stand in for another concrete type where the caller may only use the methods
declared by the specified Trait. Each possible return value from the function
must resolve to the same concrete type.

If using a generic parameter (e.g., `fn f<T: Bar>(...) -> T`), the caller
chooses the concrete type, therefore the callee must provide functions with any
return type that the caller could choose. If using `impl Trait` (e.g., `fn
f(...) -> impl Bar`), then the callee chooses the concrete type (i.e., the
compiler infers the concrete type from the function body). Therefore there is
only ever one concrete type, however, that concrete type is not known to the
caller, so the caller can only assume the trait bound.

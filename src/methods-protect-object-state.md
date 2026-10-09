# Methods Must Preserve Object State Constraints

**The core principle is that methods must preserve the object's state constraints.** If an account
promises that withdrawals cannot spend more than its available balance, its withdrawal method must
uphold that rule. If a volume control promises a level between 0 and 100, its methods must keep the
level within that range. **Preserving these rules must not depend on the caller checking first.**

A caller may query the object to decide whether to show a button, display a message, or continue a
workflow. The method that changes the state still has to enforce the object's rules when called
directly. The account and order APIs below therefore allow callers to attempt an operation even when
it cannot succeed: they reject the request and leave the state unchanged.

## The method must protect the account's balance

Suppose an account promises that a withdrawal can spend only money the account currently holds. A
caller may request any withdrawal amount, but the request is not guaranteed to succeed. If the
amount exceeds the balance, `withdraw()` must return an error and leave the balance unchanged.

Here is a small implementation, with amounts stored as whole cents:

```rust
#[derive(Debug, PartialEq, Eq)]
pub enum WithdrawError {
    InsufficientFunds,
}

pub struct Account {
    balance: u32,
}

impl Account {
    pub fn new(balance: u32) -> Self {
        Self { balance }
    }

    pub fn balance(&self) -> u32 {
        self.balance
    }

    pub fn can_withdraw(&self, amount: u32) -> bool {
        amount <= self.balance
    }

    pub fn withdraw(&mut self, amount: u32) -> Result<(), WithdrawError> {
        if !self.can_withdraw(amount) {
            return Err(WithdrawError::InsufficientFunds);
        }

        self.balance -= amount;
        Ok(())
    }
}

let mut account = Account::new(100);

assert_eq!(account.withdraw(150), Err(WithdrawError::InsufficientFunds));
assert_eq!(account.balance(), 100);

assert!(account.can_withdraw(80));

assert_eq!(account.withdraw(40), Ok(()));
assert_eq!(account.balance(), 60);

assert_eq!(account.withdraw(80), Err(WithdrawError::InsufficientFunds));
assert_eq!(account.balance(), 60);
```

The check happens before subtraction. An unsuccessful withdrawal leaves the account unchanged; a
successful withdrawal reduces the balance by the requested amount. Returning `Err` does not itself
undo mutations: the balance stays unchanged here because the method rejects the request before
modifying it.

Callers are allowed to attempt a withdrawal; success depends on the available balance. The account
owns both the decision and its enforcement: `can_withdraw()` reports whether the funds are
sufficient, and `withdraw()` uses that decision before changing the balance. Sharing the rule inside
the object keeps callers from having to reproduce it.

The choice of `u32` already excludes negative amounts and balances. [\[1\]][1] It does not establish
that `amount <= self.balance`, so the method still checks that relationship. Keeping some
nonnegative number in the field would not be enough: the account promises to subtract exactly the
requested amount on success and to keep the previous balance on rejection.

Keeping `balance` private prevents code outside the account's module from writing it directly. Code
inside that module must still respect the rule, as must every operation that changes the balance.
This is the role of encapsulation illustrated by the Rust book's `AveragedCollection`: private
fields let the type's implementation control changes and keep related values consistent. [\[2\]][2]

## The caller's `if` responds to the object's decision

A caller can use `can_withdraw()` to disable a button or decide whether to continue a workflow. The
query answers whether the account currently permits the withdrawal. The caller chooses how to react
to that answer.

The same distinction appears in
[An `if` can merely react to a domain decision](./object-coordination.md#1-an-if-can-merely-react-to-a-domain-decision).
In Vladimir Khorikov's ATM example, the application queries `CanDispenseMoney`, while
`DispenseMoney` also enforces the rule before dispensing. **The caller's `if` responds to a decision
made by the domain object; the command still protects the object's state if that query is skipped.**
Khorikov's example throws an exception, whereas this account API chooses `Result` to represent a
rejected request. [\[3\]][3]

If `withdraw()` merely subtracted the amount and expected every caller to check first, each user
interface, background job, or other entry point would become responsible for protecting the account.
A direct call that skipped the check could break the rule the account claims to enforce.

A query also does not reserve the balance. In the example above, withdrawing 80 is initially
possible, but withdrawing 40 first leaves only 60. The later request for 80 must be rejected despite
the earlier `true` result. A caller that queries first must still handle the command's result.

## Protect the object's state transitions: `Order::cancel`

Suppose an `Order` promises that a shipped order cannot be cancelled. Callers may attempt
cancellation in any state, but success is not guaranteed. Calling `cancel()` on a shipped order is
allowed: the method returns `Err(CancelError::AlreadyShipped)` and leaves the state unchanged.

In this implementation, the method checks the state before changing it:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
enum OrderStatus {
    Pending,
    Shipped,
    Cancelled,
}

#[derive(Debug, PartialEq, Eq)]
pub enum CancelError {
    AlreadyShipped,
}

pub struct Order {
    status: OrderStatus,
}

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

let mut shipped = Order {
    status: OrderStatus::Shipped,
};
assert_eq!(shipped.cancel(), Err(CancelError::AlreadyShipped));
assert_eq!(shipped.status, OrderStatus::Shipped);

let mut pending = Order {
    status: OrderStatus::Pending,
};
assert_eq!(pending.cancel(), Ok(()));
assert_eq!(pending.status, OrderStatus::Cancelled);
assert_eq!(pending.cancel(), Ok(()));
```

Blindly assigning `Cancelled` would violate the promise for a shipped order. Returning
`AlreadyShipped` without changing the state preserves it. This protection must work even when the
caller does not first call `can_cancel()`.

This is a **transition constraint**: both `Shipped` and `Cancelled` are valid status values, but
changing from the former to the latter is forbidden by this example's contract. Merely storing a
valid enum variant does not enforce that rule.

Returning `Ok(())` for an already cancelled order is another explicit design choice: repeated
cancellation is a successful no-op. A different application might choose an error. Either choice can
preserve the state constraint, provided the API documents and implements it consistently.

## References

1. Rust project contributors. “Primitive Type `u32`.” _Rust Standard Library_. Online documentation,
   accessed October 9, 2026. [Documentation][1].
2. Rust project contributors. _The Rust Programming Language_, chapter 18.1, “Characteristics of
   Object-Oriented Languages,” subsection “Encapsulation That Hides Implementation Details.” Online
   edition, accessed October 9, 2026. [Chapter][2].
3. Vladimir Khorikov. “Domain services vs Application services.” September 8, 2016. See the ATM
   query and command example in section 1. [Post][3].

[1]: https://doc.rust-lang.org/std/primitive.u32.html
[2]: https://doc.rust-lang.org/book/ch18-01-what-is-oo.html#encapsulation-that-hides-implementation-details
[3]: https://enterprisecraftsmanship.com/posts/domain-vs-application-services/

# Where Should Object Coordination Logic Live?

When designing objects, a recurring question is:

> Should the logic for calling another object be encapsulated inside the object itself, or should
> the caller coordinate those calls?

A closely related question appears in Domain-Driven Design:

> If an application service contains an `if`, does that mean domain logic has leaked into the
> application layer?

Neither question can be answered by syntax alone. A method call is not automatically misplaced
because it appears in a caller, and an `if` is not automatically domain logic because it appears in
an application service.

The real questions concern **responsibility, knowledge, collaboration, business decision-making, and
the expected direction of change**.

Responsibility-Driven Design treats an application as a community of objects whose responsibilities
include actions they perform, knowledge they maintain, and important decisions they make. Objects
collaborate because larger responsibilities usually cannot be fulfilled by one object alone.
[Wirfs-Brock][object-design] and [McKean][object-design] explicitly recognize coordinators as
legitimate object roles: a coordinator can react to events primarily by delegating tasks to other
objects. [1]

A useful starting principle is therefore:

> **An object should own the behavior necessary to fulfill its own responsibility completely. A
> caller should coordinate already meaningful capabilities when that coordination belongs to the
> caller's higher-level responsibility.**

This is a design principle rather than a mechanical rule. Determining the correct responsibility
boundary requires examining the consequences of alternative assignments.

---

## 1. Encapsulate a call when it is part of the object's own responsibility

Suppose an `Order` is responsible for supporting cancellation.

A weak interface could expose enough internal state for the caller to implement cancellation itself:

```rust
if order.status() != OrderStatus::Shipped {
    order.set_status(OrderStatus::Cancelled);
}
```

The caller now needs to know:

- which state prevents cancellation;
- which state represents cancellation;
- which transition constitutes cancellation;
- potentially which related values must also change;
- potentially which domain events or other consequences accompany cancellation.

Although the caller is invoking methods _on_ `Order`, it is effectively implementing what
cancellation means.

A stronger interface is:

```rust
order.cancel()?;
```

Now the caller knows the capability—_cancel this order_—without knowing the internal procedure
required to fulfill it.

This is closely related to the object-design principle behind _Tell, Don't Ask_: behavior that
operates on an object's state often belongs with that state rather than in clients that retrieve
information and reconstruct the behavior themselves. [Fowler][tell-dont-ask] presents _Tell, Don't
Ask_ as a useful reminder to colocate tightly coupled data and behavior, while explicitly warning
against turning it into a prohibition on query methods. [4]

The important distinction is therefore not:

```text
one call = good
several calls = bad
```

It is:

```text
caller uses the object's capability
              vs.
caller implements the object's capability
```

If every caller must understand a sequence of internal conditions and state transitions in order to
use an object correctly, the object's public abstraction is probably too weak.

---

## 2. Keeping coordination in the caller can be correct

Encapsulation does **not** mean hiding every collaboration inside one of the participating objects.

Consider:

```rust
let record = parser.parse(line)?;

if filter.accepts(&record) {
    consumer.process(record)?;
}
```

There are three potentially distinct responsibilities:

```text
Parser
    parses input

Filter
    determines whether a record satisfies a filtering policy

Consumer
    processes an accepted record
```

There is no inherent reason for `Parser` to know about the filtering policy or what eventually
happens to an accepted record.

If the larger operation is conceptually _process one input line_, introducing an object responsible
for that coordination may be appropriate:

```rust
record_processor.process(line)?;
```

whose implementation might be:

```rust
fn process(&self, line: &str) -> Result<()> {
    let record = self.parser.parse(line)?;

    if self.filter.accepts(&record) {
        self.consumer.process(record)?;
    }

    Ok(())
}
```

Here `RecordProcessor` is not stealing parsing responsibility from `Parser` or filtering
responsibility from `Filter`. Its responsibility is the higher-level coordination itself.

This fits Responsibility-Driven Design particularly well. [Wirfs-Brock][object-design] and
[McKean][object-design] define collaboration as one object requesting help from another so that the
objects can jointly fulfill larger responsibilities, and they explicitly identify **Coordinator** as
a legitimate object-role stereotype. [1]

Thus:

> **Coordination is not a design failure. Coordination can itself be a coherent responsibility.**

The question is whether the coordinating object actually represents that larger responsibility.

---

## 3. A useful test: is the caller using the object or implementing the object?

Compare:

```rust
order.cancel()?;
```

with:

```rust
if order.status() != OrderStatus::Shipped {
    order.set_status(OrderStatus::Cancelled);
}
```

In the first version, the caller is using an `Order`.

In the second, the caller partially implements order cancellation.

This gives a practical diagnostic:

> **Does the caller need to know how the collaborator fulfills its responsibility, or only what
> capability it provides?**

Warning signs include callers needing to know:

- internal state representations;
- implementation-specific conditions;
- several state mutations that must remain synchronized;
- mandatory internal steps;
- cleanup details;
- domain rules that are supposed to define the collaborator's own behavior.

For example:

```rust
account.debit(amount)?;
account.reserve(amount)?;
account.record_transfer(target)?;
account.raise_transfer_started()?;
```

If every client initiating a transfer must perform exactly those operations in exactly that way, the
API may be exposing implementation fragments rather than a meaningful operation.

A more appropriate interface might be:

```rust
account.start_transfer(target, amount)?;
```

provided that _starting a transfer_ really is the responsibility of that object.

The last qualification matters. Moving code inward is not automatically better.

---

## 4. The opposite mistake: responsibility inflation

Suppose the earlier code:

```rust
let record = parser.parse(line)?;

if filter.accepts(&record) {
    consumer.process(record)?;
}
```

is "encapsulated" as:

```rust
parser.parse_filter_and_consume(line)?;
```

Now `Parser` may need to know:

- the filtering policy;
- the consumer;
- what to do with rejected records;
- whether processing errors abort a batch;
- perhaps persistence;
- perhaps notifications;
- perhaps retry policy.

Nothing has necessarily become better encapsulated. Instead, the parser's responsibility has
expanded from _parsing_ into _running the surrounding workflow_.

So the opposite diagnostic is:

> **If I move this coordination into the object, does that make its existing responsibility more
> complete, or does it merely make the object know more unrelated things?**

Compare:

```text
Order
└── knows how to cancel itself correctly
```

with:

```text
Parser
├── parses
├── filters
├── persists
├── retries
├── notifies
└── manages batch policy
```

The first strengthens a responsibility.

The second accumulates responsibilities.

Responsibility-Driven Design explicitly warns against both extremes: excessively centralized control
can produce objects that merely obey a controller, while excessively dispersed control can lead to
weak objects and awkward collaborations. The appropriate control style depends on the problem and
the neighborhood of collaborating objects. [1]

---

## How to Review Coordination Logic

When deciding whether logic belongs inside an object or in its caller, the following questions are
more useful than rules such as "avoid `if`" or "hide all method sequences."

### 1. Whose responsibility is being fulfilled?

Start by describing the code in domain or design language rather than in terms of methods.

For example:

```rust
let record = parser.parse(line)?;

if filter.accepts(&record) {
    consumer.process(record)?;
}
```

may correspond to three responsibilities:

```text
parse a line
determine whether a record is accepted
process an accepted record
```

Then ask whether the whole operation itself represents another meaningful responsibility:

```text
process one input line
```

If so, a coordinating object can legitimately own the collaboration.

If, instead, the code consists of several low-level steps that every client must perform merely to
complete _one collaborator's own advertised responsibility_, those steps probably belong behind that
collaborator's abstraction.

---

### 2. Does the caller know implementation details?

Ask:

> What must the caller know in order to use this object correctly?

A caller knowing that an order can be cancelled is normal.

A caller knowing:

```text
check state A
then modify field B
then clear field C
then emit event D
```

may indicate that cancellation behavior has leaked outward.

But do not classify all required ordering as implementation leakage. An ordering constraint may be
an intentional part of an object's contract. Object contracts explicitly include conditions under
which operations may be used and, in some cases, required ordering. [1]

The distinction is whether that protocol is an intentional public contract or merely an exposed
internal procedure that callers have to reconstruct.

---

### 3. Would moving the logic inward broaden the object's role?

Now reverse the question:

> If this coordination is moved into the object, what new concepts must that object understand?

If `Parser` must suddenly know about:

```text
filtering
persistence
notifications
retries
transactions
batch policy
```

to "encapsulate" a workflow, that is evidence that `Parser` is the wrong owner.

The right location is not necessarily the deepest possible object.

It is the object whose role makes the responsibility coherent.

---

### 4. Which placement keeps expected changes within the right boundary?

This is one of the most useful practical tests.

It is a **design heuristic derived from responsibility and collaboration analysis**, not a formal
DDD law.

Consider again:

```rust
let record = parser.parse(line)?;

if filter.accepts(&record) {
    consumer.process(record)?;
}
```

We can simulate several expected changes.

#### Parsing format changes

Suppose:

```text
CSV
 ↓
TSV
```

or:

```text
record format v1
       ↓
record format v2
```

This should primarily affect:

```text
Parser
```

The surrounding workflow should not have to understand delimiter rules, tokenization, escaping,
quoting, or field extraction.

If changing the input representation requires modifying every caller of `Parser`, parsing details
have probably leaked outside the parser abstraction.

#### Filtering policy changes

Suppose the original rule is:

```text
accept score >= 80
```

and later becomes:

```text
accept score >= 80
AND category is not excluded
```

That change should primarily affect:

```text
Filter
```

The parser should not need to change.

The consumer should not need to know how the selection rule is calculated.

If callers instead contain:

```rust
if record.score() >= 80 && !record.is_excluded() {
    consumer.process(record)?;
}
```

then the filtering policy has escaped its intended boundary.

#### Processing workflow changes

Suppose the original flow is:

```text
accepted
   ↓
process immediately
```

and later becomes:

```text
accepted
   ↓
validate
   ↓
persist
   ↓
notify
```

or:

```text
accepted
   ↓
enqueue for asynchronous processing
```

That change should primarily affect the workflow coordinator and the relevant processing
collaborators.

It should not require modifying `Parser`.

It should not require modifying how `Filter` determines acceptance.

The expected change topology is therefore approximately:

```text
Parsing format changes
        ↓
      Parser


Filtering policy changes
        ↓
      Filter


Processing workflow changes
        ↓
 RecordProcessor / Consumer
```

If a parsing-format change forces modifications in all callers, parsing details may have leaked
outward.

If a processing-workflow change forces modifications to `Parser`, `Parser` may have absorbed a
responsibility that belongs to the surrounding workflow.

However, the goal is **not to minimize the raw number of files changed**.

Suppose everything is placed in:

```text
Parser
├── parsing
├── filtering
├── persistence
├── notifications
└── retry behavior
```

Then a parsing change, filtering change, persistence change, and notification change may each touch
only one class.

That does not mean the design has good change locality.

It means unrelated reasons for change have been concentrated into the same object.

The stronger criterion is:

> **Does a change primarily affect the part of the design responsible for the concept being
> changed?**

In other words:

> **Related changes should tend to stay together; unrelated changes should not be forced together
> merely to reduce the number of modified files.**

This reasoning is consistent with _Object Design_. In Chapter 4, when discussing difficulty
assigning a responsibility, [Wirfs-Brock][object-design] and [McKean][object-design] recommend
trying an assignment and then examining what that choice implies for surrounding objects. Their
concrete example asks whether a `Session` should time itself or whether `SessionManager` should do
so. Both alternatives can be made workable; the proposed way forward is to examine how each
responsibility distribution affects neighboring objects. The example appears on p. 139. [1]

That example is important because it rejects a simplistic view of responsibility assignment:

```text
find universal rule
        ↓
derive unique owner
```

Instead, responsibility assignment is often:

```text
candidate responsibility
        ↓
try plausible owner
        ↓
trace collaborations and knowledge
        ↓
consider expected changes and consequences
        ↓
compare alternatives
```

The parser/filter/processor analysis above is an application of that reasoning. It is not a
quotation or named rule from the book.

---

### 5. Who makes the business decision?

For application services in particular, one additional question matters:

> **Is the application service executing a decision already made by the domain, or is it deriving a
> new business decision from domain information?**

This is the key to understanding why an `if` in an application service is not automatically a
problem.

[Evans][domain-driven-design] describes the Application Layer as defining the jobs the software
performs and coordinating domain objects, while keeping business rules and business knowledge in the
Domain Layer. [2] [Fowler][service-layer] and [Stafford][service-layer] similarly distinguish
**application logic**, which coordinates an application's response to a use case, from **domain
logic**, which expresses rules of the problem domain. [3]

The important word is **coordinates**.

Coordination naturally contains control flow.

---

## `if` Statements in Application Services

### 1. An `if` can merely react to a domain decision

[Vladimir Khorikov][domain-vs-application-services] gives a particularly clear example in _Domain
services vs Application services_. Consider the equivalent Rust-style code:

```rust
if !atm.can_dispense_money(amount) {
    return Ok(());
}

atm.dispense_money(amount)?;
```

The application service contains an `if`.

Nevertheless, according to [Khorikov][domain-vs-application-services]'s analysis, the application
service is not making the withdrawal decision. `Atm` determines whether money can be dispensed; the
application service merely decides whether to continue the use-case flow after receiving that
answer. [6]

Semantically:

```text
Domain:
    "This ATM cannot dispense this amount."

Application service:
    "Then this use case stops here."
```

The first statement is domain knowledge.

The second is orchestration.

[Khorikov][domain-vs-application-services] additionally requires `DispenseMoney` itself to preserve
the ATM invariant rather than relying on the outer `CanDispenseMoney` check as its only protection.
[6]

The caller's `if` responds to a decision made by the domain object; the command still protects the
object's state if that query is skipped. For a rule the object promises to maintain, callers must
not become its only enforcement point. See
[Methods Must Preserve Object State Constraints](./methods-protect-object-state.md) for account and
order examples.

---

### 2. Another `if` can encode a domain rule

[Khorikov][domain-vs-application-services] then changes the scenario.

Suppose charging the customer can fail:

```rust
let amount_with_commission =
    atm.calculate_amount_with_commission(amount);

let payment = payment_gateway
    .charge(amount_with_commission)
    .await?;

if payment.is_failed() {
    return Ok(());
}

atm.dispense_money(amount)?;
```

Now assume the business rule is:

> Cash must not be dispensed unless payment succeeds.

The second `if` is different.

The application service is connecting two facts:

```text
payment failed
      ↓
cash must not be dispensed
```

That relationship is itself part of the business decision.

[Khorikov][domain-vs-application-services] therefore classifies this branch as domain logic: unlike
the first `if`, the decision is no longer being made by `Atm`; the application service itself
determines the business consequence of the payment result. [6]

This example gives a much better test than cyclomatic complexity:

> **The existence of a branch does not identify domain logic. The semantic decision expressed by the
> branch does.**

[Khorikov][domain-vs-application-services]'s proposed solution in this particular scenario is a
Domain Service that participates in the payment-dependent decision. He calls a Domain Service that
depends on an external gateway an **"impure domain service."** That terminology and recommendation
are [Khorikov][domain-vs-application-services]'s own formulation; "impure domain service" is not a
canonical [Evans][domain-driven-design] DDD term. [6]

The broader, well-supported DDD principle is that a Domain Service is appropriate when an important
domain operation does not naturally belong to an Entity or Value Object.
[Evans][domain-driven-design] describes such services as domain operations expressed in terms of the
domain model, and [Vernon][implementing-ddd] gives the same general guidance. [2][5]

Exactly how a business decision that depends on external information should be modeled can depend on
consistency requirements, transaction boundaries, integration design, and the domain itself. A
Domain Service is one option, not an automatic consequence of seeing an external call.

---

## Domain methods alone do not prove that no rule has leaked

Consider:

```rust
if customer.is_premium() {
    order.apply_discount(Discount::TwentyPercent)?;
}
```

At first glance, both operations appear properly domain-oriented:

```rust
customer.is_premium()
order.apply_discount(...)
```

But inspect the entire branch:

```text
IF customer is premium
THEN apply a 20% discount
```

The branch expresses a further rule:

> Premium customers receive a 20% discount.

`Customer` may own the definition of premium status.

`Order` may own the mechanics and invariants of applying a discount.

But the relationship:

```text
PremiumCustomer → TwentyPercentDiscount
```

may belong to neither object individually.

It may represent a separate pricing policy.

A possible design is:

```rust
if let Some(discount) =
    pricing_policy.discount_for(&customer, &order)
{
    order.apply_discount(discount)?;
}
```

Here the meaning of `discount_for` is crucial.

If it means:

> "According to the pricing policy, this is the discount that applies to this order."

then the policy has already made the domain decision. The application layer is reacting to its
result.

If it means only:

> "Here is one possible discount that the customer might be eligible to request."

then the application service may still be deciding whether the discount should actually be applied.

Therefore:

> **Changing `bool` into `Option<T>` or moving individual calculations behind domain methods does
> not automatically improve responsibility allocation.**

The semantic contract matters.

This condition-to-consequence analysis is a derived design heuristic from the preceding principles;
it is not a named [Evans][domain-driven-design] or [Wirfs-Brock][object-design] rule.

---

## Domain logic can hide in combinations of domain predicates

Consider:

```rust
if customer.is_premium()
    && order.meets_minimum_amount()
{
    order.apply_discount(discount)?;
}
```

Suppose both predicates are correctly implemented in domain objects.

The application service still expresses:

> A discount applies only when both conditions are true.

The business policy may therefore reside in the `&&`.

The same issue can appear with:

```rust
if a || b { ... }
```

or:

```rust
if a && !b { ... }
```

or:

```rust
if a {
    ...
} else if b {
    ...
}
```

Each individual fact can come from the domain while the **composition of those facts into a
decision** remains outside it.

That does not mean every boolean expression must become a Domain Service. It means that the
expression should be examined semantically.

Ask:

> Is this merely application control flow, or does the combination of conditions define a business
> policy?

---

## Domain logic can also hide in call ordering

The same issue exists without any `if`.

Consider:

```rust
payment.capture()?;
order.ship()?;
```

A sequence of calls is not automatically domain logic.

But suppose the business rule is:

> An unpaid order must never be shipped.

If the only thing protecting this rule is that one application service happens to invoke `capture()`
before `ship()`, the business constraint may be encoded implicitly in orchestration.

In a DDD model, if "paid before shipping" is an invariant belonging to the `Order` Aggregate,
`order.ship()` should not allow a transition that violates that invariant. Aggregate design treats
true invariants as part of the Aggregate's consistency boundary. [5]

But again, not every required method ordering represents a domain invariant.

An ordering may instead be:

- an infrastructure constraint;
- a protocol requirement;
- transaction management;
- application workflow;
- a documented object-contract precondition.

That is why the meaning of the sequence must be examined rather than the syntax.

---

## Facts, decisions, and orchestration

A useful mental model is to separate three things.

#### Domain facts

Examples:

```text
Customer is premium.
Order total is $200.
Payment failed.
Inventory contains four units.
```

#### Domain decisions

Examples:

```text
This customer receives a 20% discount.
This order may be cancelled.
This withdrawal is permitted.
This shipment requires manual approval.
```

#### Application orchestration

Examples:

```text
Load an aggregate.
Call a domain operation.
Call an external gateway.
Persist the result.
Publish an integration message.
Translate a domain failure into a use-case result.
```

[Evans][domain-driven-design]'s distinction between Application and Domain Layers, and
[Fowler][service-layer]/[Stafford][service-layer]'s distinction between application logic and domain
logic, support this separation conceptually, although the exact three-part classification above is a
practical synthesis rather than terminology defined by those authors. [2][3]

A common leakage pattern is:

```text
domain fact
    ↓
application service interprets it
    ↓
new domain decision
```

For example:

```rust
if customer.is_premium() {
    discount = Discount::TwentyPercent;
}
```

The important issue is not the `if`.

It is that the application service has interpreted a domain fact to produce a pricing decision.

By contrast:

```rust
let decision =
    pricing_policy.discount_for(&customer, &order);
```

can represent the domain making the pricing decision, provided that this is genuinely the policy
object's responsibility.

---

## Application services are allowed to coordinate multiple domain calls

Another common overcorrection is to assume:

> "If an application service calls two or three domain methods, those calls must be wrapped in a
> Domain Service."

That is not supported by classical DDD.

[Evans][domain-driven-design]'s Application Layer explicitly coordinates domain objects.
[Fowler][service-layer] and [Stafford][service-layer]'s Service Layer likewise coordinates an
application's response to a use case while delegating domain logic to the Domain Model. [2][3]

[Khorikov][domain-vs-application-services] makes the same point with his ATM example: merely knowing
that two domain operations must be invoked does not by itself constitute domain knowledge. His
additional heuristic is that if changing the ordering does not affect domain invariants, that is
evidence that the sequence is orchestration rather than an exposed domain rule. [6]

For example:

```rust
atm.dispense_money(amount)?;
let commission = atm.calculate_commission(amount);
```

may be perfectly reasonable application-level coordination if each operation already has meaningful
domain semantics and their sequence does not encode another business policy.

Therefore:

> **Multiple domain calls are not evidence of leakage.**

The question remains:

> What decision, if any, is the application service making between those calls?

---

## A coordinator should not automatically become a Domain Service

A useful distinction is:

```text
coordination
≠
domain service
```

Responsibility-Driven Design recognizes **Coordinator** as a general object role. [1]

DDD's **Domain Service** is more specific. [Evans][domain-driven-design] introduces a Domain Service
for an operation that is an important domain concept but does not naturally belong to an Entity or
Value Object. [2] [Vernon][implementing-ddd] repeats essentially the same criterion: a Domain
Service is appropriate when a domain-specific operation feels misplaced on an Aggregate or Value
Object. [5]

Therefore an object such as:

```text
RecordProcessor
```

that coordinates parsing, filtering, persistence, or application workflow is not automatically a
Domain Service.

Its architectural placement depends on what its responsibility means.

If it expresses:

```text
process this application's import request
```

it may be an application-level coordinator.

If it expresses a genuine domain operation:

```text
determine legally valid settlement between these accounts
```

and that operation does not naturally belong to one Entity or Value Object, a Domain Service may be
appropriate.

Naming something `Service` or `Processor` does not determine its layer.

Its responsibility does.

---

## _Tell, Don't Ask_ is a heuristic, not a ban on queries

The discussion above can easily be distorted into:

> "Never ask an object for information. Always tell it to do everything."

[Fowler][tell-dont-ask] explicitly rejects that extreme interpretation. He notes that objects can
collaborate effectively by providing information and that eliminating reasonable query methods can
produce unnecessarily convoluted designs. He treats _Tell, Don't Ask_ mainly as a prompt to consider
colocating behavior with the data it depends on, not as an absolute law. [4]

Therefore this is not inherently wrong:

```rust
if filter.accepts(&record) {
    consumer.process(record)?;
}
```

`Filter` may be an information-producing collaborator whose responsibility is precisely to answer
whether a record satisfies a policy.

The important question is whether the caller is merely using that answer for its own responsibility
or whether it is reconstructing additional domain policy from it.

---

## When these guidelines should not be applied mechanically

These ideas are most useful when there is meaningful domain behavior to model.

They should not be used to force every program into a rich object model.

[Khorikov][domain-vs-application-services] explicitly notes that simple CRUD applications may
contain no substantive domain decision-making at all, in which case application services can perform
the workflow directly and a rich domain model may provide little benefit. [6]

Likewise, [Fowler][service-layer]'s Domain Model is one architecture pattern among others; he does
not claim it is always superior to Transaction Script. [3][4]

Other cases where a more direct design may be preferable include:

- straightforward data transformation with little business semantics;
- thin integration layers;
- technical pipelines where the sequencing is infrastructural rather than domain-driven;
- deliberately functional architectures where responsibility is organized through functions and
  modules rather than stateful objects.

The underlying questions about cohesion, knowledge, and change boundaries still apply, but the
object-oriented vocabulary may not be the best representation.

---

## A Practical Review Procedure

When a piece of coordination logic feels questionable, review it in this order.

#### Step 1: State the responsibility in plain language

Do not start with:

```text
Parser calls Filter, then calls Consumer.
```

Start with:

```text
Parse an input record.
Determine whether the record is accepted.
Process accepted records.
```

Then ask whether there is also a meaningful larger responsibility:

```text
Process one input line.
```

#### Step 2: Identify who owns each decision

Ask:

```text
Who decides whether the record is acceptable?

Who decides what an accepted record means?

Who decides what should happen after acceptance?

Who decides what happens after rejection?
```

Do not assume that all of these belong to the same object.

#### Step 3: Check knowledge leaking outward

Ask:

> What details must callers know in order to use this collaborator correctly?

If clients repeatedly know internal state, internal transitions, or internal implementation
sequences, the collaborator may not own its responsibility completely.

#### Step 4: Check knowledge leaking inward

Ask:

> What unrelated concepts would this object need to understand if I moved the coordination inside
> it?

If `Parser` suddenly needs knowledge of databases, retry policy, notifications, and batch workflows,
moving the code inward is probably making the abstraction worse.

#### Step 5: Simulate likely changes

Ask:

```text
If parsing changes, who changes?

If filtering changes, who changes?

If the workflow changes, who changes?
```

Then compare the actual propagation of change with the intended responsibility boundaries.

Do not merely count modified classes.

Ask whether the modified classes are the ones responsible for the changed concept.

#### Step 6: For an application-service branch, translate the whole branch into business language

For:

```rust
if customer.is_premium() {
    order.apply_discount(Discount::TwentyPercent)?;
}
```

write:

> Premium customers receive a 20% discount.

That exposes the hidden pricing policy.

For:

```rust
if let Err(reason) = order.cancel() {
    return Ok(CancelOutcome::Rejected(reason));
}
```

write:

> If the domain rejects cancellation, return a rejected use-case result.

That looks much more like orchestration.

#### Step 7: Check invariants separately from orchestration

Finally ask:

> If another caller bypassed this application-service branch, could the domain object enter a state
> it is responsible for forbidding?

If yes, an invariant may be insufficiently protected.

But do not use this as the _only_ leakage test. A pricing rule can be leaked even when every
resulting `Order` state is technically valid.

For example, an order may legally support any discount between 0% and 100%. That does not mean:

> Premium customers receive 20%.

is not domain knowledge.

Invariant protection and policy placement are related but distinct questions.

---

## Final Principle

The most useful question is not:

> "Should this method call be inside the object?"

Nor:

> "Is an `if` allowed in an application service?"

Nor:

> "Can I make only one class change?"

Instead ask:

> **Whose responsibility does this logic implement?**

Then test the proposed answer from four directions:

```text
Responsibility
    Does this behavior complete the object's actual role?

Knowledge
    Does either placement force an object to know things outside that role?

Change
    When an expected requirement changes, does the change land where
    that concept is owned?

Decision
    Is the caller merely coordinating an existing decision, or is it
    creating a new business decision from lower-level facts?
```

A well-factored design tends toward:

```text
Domain object / policy
    owns the rules and decisions that belong to it
                ↓
Coordinator
    combines meaningful capabilities into a larger responsibility
                ↓
Application service
    orchestrates the use case and interactions with the outside world
```

But these are responsibility boundaries, not mandatory class layers.

The decisive distinction is:

> **Objects should own enough behavior to fulfill their responsibilities without forcing clients to
> implement them. At the same time, an object should not absorb a surrounding workflow merely to
> hide collaboration.**

And for DDD application services:

> **Control flow is allowed. Domain decision-making is what must be examined.**

The presence of an `if`, multiple method calls, a query method, or a coordinator is only syntax and
structure.

The design question lies in the meaning behind them.

### References

[1] [Rebecca Wirfs-Brock][object-design] and [Alan McKean][object-design], _Object Design: Roles,
Responsibilities, and Collaborations_. Addison-Wesley Professional, 2003. Particularly relevant are
Chapter 1, “Design Concepts”; Chapter 4, “Responsibilities”; Chapter 5, “Collaborations”; and
Chapter 6, “Control Style.” The `Session` versus `SessionManager` responsibility-assignment example
is in Chapter 4, p. 139.
[Publisher page](https://www.informit.com/store/object-design-roles-responsibilities-and-collaborations-9780201379433?utm_source=chatgpt.com)

[2] [Eric Evans][domain-driven-design], _Domain-Driven Design: Tackling Complexity in the Heart of
Software_. Addison-Wesley Professional, 2003. See Chapter 4, “Isolating the Domain,” for the
Application/Domain/Infrastructure layer distinction, and Chapter 5, “A Model Expressed in Software,”
for Services.
[Book page and table of contents](https://www.oreilly.com/library/view/domain-driven-design-tackling/0321125215/?utm_source=chatgpt.com)

[3] [Martin Fowler][service-layer] with [Randy Stafford][service-layer], _Patterns of Enterprise
Application Architecture_. Addison-Wesley, 2002, Chapter 9, “Domain Logic Patterns,” Service Layer.
The Service Layer material distinguishes domain logic from application/workflow logic and describes
application services as coordinating use-case responses while delegating domain logic to the Domain
Model.
[Service Layer excerpt](https://www.martinfowler.com/eaaCatalog/serviceLayer.html?utm_source=chatgpt.com)

[4] [Martin Fowler][tell-dont-ask], “_Tell Don’t Ask_,” 2013. [Fowler][tell-dont-ask] explains the
value of colocating behavior with data while explicitly warning against treating _Tell, Don’t Ask_
as a rule forbidding query methods; he also notes that layering and other concerns can outweigh
strict colocation. [Article](https://martinfowler.com/bliki/TellDontAsk.html?utm_source=chatgpt.com)

[5] [Vaughn Vernon][implementing-ddd], _Implementing Domain-Driven Design_. Addison-Wesley
Professional, 2013. See Chapter 7, “Services”; Chapter 10, “Aggregates”; and Chapter 14,
“Application.” [Vernon][implementing-ddd] describes Domain Services as domain-specific operations
that do not naturally fit an Aggregate or Value Object, Aggregates as consistency boundaries for
true invariants, and Application Services as coordinators of use-case tasks.
[Book page](https://www.informit.com/store/implementing-domain-driven-design-9780133039900?utm_source=chatgpt.com)

[6] [Vladimir Khorikov][domain-vs-application-services], “Domain services vs Application services,”
_Enterprise Craftsmanship_, September 8, 2016. This article provides the ATM examples distinguishing
an application-service `if` that reacts to an `Atm` decision from an `if` that makes a business
decision based on payment failure. Its “impure domain service” terminology and the recommendation
for that specific external-dependency scenario are [Khorikov][domain-vs-application-services]'s
formulation rather than terminology defined by [Evans][domain-driven-design].
[Article](https://enterprisecraftsmanship.com/posts/domain-vs-application-services/?utm_source=chatgpt.com)

[object-design]: https://www.informit.com/store/object-design-roles-responsibilities-and-collaborations-9780201379433?utm_source=chatgpt.com
[domain-driven-design]: https://www.oreilly.com/library/view/domain-driven-design-tackling/0321125215/?utm_source=chatgpt.com
[service-layer]: https://www.martinfowler.com/eaaCatalog/serviceLayer.html?utm_source=chatgpt.com
[tell-dont-ask]: https://martinfowler.com/bliki/TellDontAsk.html?utm_source=chatgpt.com
[implementing-ddd]: https://www.informit.com/store/implementing-domain-driven-design-9780133039900?utm_source=chatgpt.com
[domain-vs-application-services]: https://enterprisecraftsmanship.com/posts/domain-vs-application-services/?utm_source=chatgpt.com

- Feature Name: `const_trait_impls`
- Start Date: 2026-07-04
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

## Summary
[summary]: #summary

This RFC makes the associated functions of traits callable in const contexts.  Inside `const`-marked items, trait bounds are conditionally const by default: const when called from a const context.  Each function's generic type parameters also carry an implied conditionally const `Destruct` (const-droppability) bound.  These defaults apply in every edition inside `const trait` and `const impl`; inside `const fn`, they apply beginning with the next edition.

Everything the defaults do has an explicit spelling that means the same thing in every edition: `T: ~const Tr` is the conditionally const bound, `T: const Tr` is the always-const bound, and `T: do() Tr` opts a bound out of the default.  The `~const` and `const` spellings are sugar for a general form, `do(~const)` and `do(const)`.

This RFC adopts the core semantic model of [RFC 3762].  It provides a full specification and answers the remaining questions: how the bounds are spelled and what the defaults are.

[RFC 2632]: https://github.com/rust-lang/rfcs/pull/2632
[RFC 3762]: https://github.com/rust-lang/rfcs/pull/3762

## Motivation
[motivation]: #motivation

### The capability

We added `const fn` in Rust 1.31.  Since then, we've made more and more of the language work in constant evaluation.  But we still can't call a trait's associated functions in const contexts: `const fn max<T: Ord>` can be written today, but its body can't compare two `T`s.  The standard library contains hundreds of functions that are otherwise const-eligible.

### The stakes: this syntax will be everywhere

Once traits work in const contexts, well-written libraries will constify nearly everything that can be constified.  Early signs that conditionally const bounds will be pervasive include:

- The standard library already contains at least 76 `const trait`s and 773 `const impl`s ([the library team's April 2026 accounting][rust-lang/rust#155816]), with 474 occurrences of the conditionally const marker (163 of them on `Destruct`) as of 2026-07-02 — and `Iterator` is only partially converted.
- In a [2026 survey][const-traits-survey] of Rust developers (N = 48, including maintainers of top-1000 crates), 26 of 31 respondents expect the conditionally const bound to be the most-used bound form in practice.
- The constified functions in the `cmp`, `Option`, `Result`, and `array` families needed conditional constness on every bound (except `Copy` bounds and some sizedness bounds, both discussed later).

[rust-lang/rust#155816]: https://github.com/rust-lang/rust/issues/155816
[const-traits-survey]: https://cel.cs.brown.edu/const-traits-analysis/

Consider this generic function:

```rust
struct Vec2<T> { x: T, y: T }

// (1) Today: generic math, runtime only.
impl<T> Vec2<T> {
    pub fn lerp(self, rhs: Self, t: T) -> Self
    where
        T: Add<Output = T> + Sub<Output = T> + Mul<Output = T> + Copy,
    {
        Vec2 {
            x: self.x + (rhs.x - self.x) * t,
            y: self.y + (rhs.y - self.y) * t,
        }
    }
}
```

If we were to write the needed conditionally const bounds explicitly, it'd look like this:

```rust
// (2) With the conditionally const bounds written explicitly.
const impl<T> Vec2<T> {
    pub fn lerp(self, rhs: Self, t: T) -> Self
    where
        T: ~const Add<Output = T>
            + ~const Sub<Output = T>
            + ~const Mul<Output = T>
            + Copy,
    { ... }
}
```

Under this RFC, we instead just add a `const` in front:

```rust
// (3) Under this RFC, in every edition.
const impl<T> Vec2<T> {
    pub fn lerp(self, rhs: Self, t: T) -> Self
    where
        T: Add<Output = T> + Sub<Output = T> + Mul<Output = T> + Copy,
    { ... }
}
```

## Design axioms
[design-axioms]: #design-axioms

Six axioms drive the choices in this RFC.

1. **Const is coming everywhere.**  Most generic items in well-written Rust will be const, and the conditionally const bound will be the most common bound modifier in the language.  *Consequence*: design the syntax for ubiquity, not novelty.

2. **The common case pays nothing; the rare case pays a little; everything can be written explicitly.**  The `Sized` default, lifetime elision, and automatic capturing of generics in return-position `impl Trait` (see [RFC 3498] and [RFC 3617]) are success stories.  *Consequence*: the conditional bound becomes implicit on `const`-marked items, with an explicit opt-out and explicit spellings.

3. **`const` always means const.**  Bare `const` on bounds means unconditionally const, preserving the parity between `const` bounds and `const` blocks: a `const {}` block may call a bound's associated functions only when the bound says `const`.  *Consequence*: `T: const Tr` is the always-const bound; we don't overload bare `const` as the conditional form.

4. **Keep doors open.**  Const is the first effect-like qualifier to get bound-level genericity, and the floor of the effect space is open: a future qualifier such as `nopanic` could lower it.  *Consequence*: make extensible choices for the grammar; include an opt-out that recovers the default rather than naming the bottom.

5. **Infer on bounds, not types.**  Defaults and elision apply only to bounds.  *Consequence*: `dyn Tr` and `fn()` pointer types take no modifiers and stay explicit; the bounds of return-position `impl Trait` are bounds (the item bounds of an anonymous output type) and elide like the rest.

6. **Prefer inter-edition consistency over intra-edition consistency (from [RFC 3085]).**  When we're moving toward a bright shiny future, we'll accept some *intra-edition* tension so that more things work the same way *across* editions.  We did this in [RFC 3498], accepting that RPIT lifetime capture would work differently in trait impls than in free functions in pre-2024 editions.  *Consequence*: `const trait` and `const impl` interiors behave identically in every edition; the edition boundary migrates only functions declared `const fn`.

**Non-goals.**  This RFC does not propose per-function constness, a general effect system, const trait objects, or const function pointers; does not rename `const fn` or `const {}` blocks; and does not change what `const fn` means for its callers.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

First we'll describe how the feature works in the next edition, then the behavior in existing editions and what to expect during migration.

### Const traits and const impls

A trait opts in to constant evaluation by adding `const` to its declaration.  An impl opts in by doing the same:

```rust
const trait Hash {
    fn hash(&self) -> u64;
}

struct Fnv(u64);

const impl Hash for Fnv {
    fn hash(&self) -> u64 { self.0 }
}

// The `const impl` is what makes this valid:
const FNV_SEED: u64 = Fnv(0xcbf29ce484222325).hash();
```

Marking a trait `const` says that impls of the trait may promise that the associated functions work in constant evaluation.  Marking an impl `const` says that the associated functions *do* work in constant evaluation.  Default function bodies in a `const trait` definition must work in constant evaluation (as if they had been written in a `const impl`).  A non-`const` impl can be written for a const trait: the functions can then be called at runtime but not in constant evaluation.

`const` in front of an item puts that whole item in the const world.

For three classes of traits that can never have associated functions (auto traits, `#[marker]` traits, and `Copy`), an ordinary impl is enough to allow use in const contexts.

### Bounds in `const`-marked items

Consider this generic function you might write today:

```rust
fn max_by_key<T, K: Ord, F: Fn(&T) -> K>(a: T, b: T, f: F) -> T {
    if f(&b) >= f(&a) { b } else { a }
}
```

To make it callable in a const context, you just add `const` in front:

```rust
const fn max_by_key<T, K: Ord, F: Fn(&T) -> K>(a: T, b: T, f: F) -> T {
    if f(&b) >= f(&a) { b } else { a }
}
```

Under the hood, this desugars to:

```rust
const fn max_by_key<T, K, F>(a: T, b: T, f: F) -> T
where
    T: do(~const) Destruct,
    K: do(~const) Ord + do(~const) Destruct,
    F: do(~const) Fn(&T) -> K + do(~const) Destruct,
{
    if f(&b) >= f(&a) { b } else { a }
}
```

The `do(~const) Tr` bound says that the trait's associated functions can be called when the function is called in a const context.  The `do(~const) Destruct` bound says that values of the type can be dropped when the function is called in a const context.  These bounds can also be written as `~const Tr` and `~const Destruct`.  A const-context caller passing a callable also needs the callable's own `Fn` impl to be const: `const fn`s have one; closures will get one through `const ||` syntax, which is future work.  Until then, a `const fn` that returns a closure opts its return bound out with the `do()` opt-out below.

On each `const fn`, `const trait`, and `const impl`, and on each function within a `const trait` or `const impl`, every trait bound written without an explicit modifier (`do(..)`, `~const`, or `const`) gets `do(~const)` automatically — including supertrait bounds, associated-type item bounds, and the bounds of return-position `impl Trait`.

Each generic type parameter of a `const fn` or of an associated function within a `const trait` or `const impl` gets an automatic `do(~const) Destruct` bound (including the implicit parameters created by argument-position `impl Trait`) unless an explicit `Destruct` bound is present.  Write `T: do() Destruct` to decline it or `T: const Destruct` to strengthen it.

The automatic `do(~const)` on bounds does not extend to the headers of non-`const` impls (even when they contain `const fn`s), to function pointer types, or to `dyn Tr`.  And the automatic `Destruct` bound attaches only to the generic type parameters of functions — not to associated types, not to the opaque type of a return-position `impl Trait`, and not to `Self` or the other generic parameters of a `const trait` or `const impl` header.

### When you need *always*-const: `T: const Tr`

A `const {}` block runs in constant evaluation no matter where it appears, even inside a function being called at runtime.  A conditionally const bound isn't enough there; you need the promise unconditionally:

```rust
const fn precomputed<T: const Default>() -> T {
    const { T::default() } // Runs in constant evaluation, always.
}
```

### When you need *less*: the `do()` opt-out

Occasionally a bound in a const function isn't there for its associated functions, e.g., because you only store a value of the type or only want the trait's associated types or constants.

```rust
const fn buffer_len<T: do() Serialize>() -> usize {
    T::MAX_LEN // Uses an associated const; calls no associated function.
}
```

The `do()` in `T: do() Serialize` resets the bound to a plain one without any const obligation, allowing even const-context callers to use a type with a non-`const` `Serialize` impl.

Read `do()` as "exactly these effect modifiers apply: none".  It's the same shape as `+ use<>` precise capturing ([RFC 3617]), which says "exactly these generics are captured: none".  Each overrides an implicit default with an exact list, and each should be needed only rarely.

Why empty parentheses rather than a word such as `do(runtime)`?  There's no good word to put there.  What goes inside `do(..)` mirrors the qualifier position on `fn` items (`do(const)` is to a bound what `const` is to an `fn`), and the empty list matches the unqualified `fn`, which carries no restriction.  The opt-out recovers that default without naming it.  The Rationale returns to this.

### Dropping values

Dropping a value in constant evaluation runs its drop glue there, so const code needs to know that values of its generic types can be dropped that way.  That property is *const* `Destruct`: plain `Destruct` is a marker trait that every type implements, and a type is const `Destruct` when its drop glue can run in constant evaluation (it has no drop glue, or its `Drop` impl is a `const impl` and its components are recursively const `Destruct`).

You will rarely write it for a parameter: the automatic `do(~const) Destruct` bound above already covers every generic type parameter of a function.  So this compiles:

```rust
const fn filter_pair<T, P: Fn(&T) -> bool>(a: T, b: T, p: P) -> Option<T> {
    if p(&a) { Some(a) } else if p(&b) { Some(b) } else { None }
    // Whichever of `a` and `b` isn't returned is dropped here.  That's
    // OK: `T` is conditionally const `Destruct` by default.
}
```

And when a function wants to accept types that *can't* be dropped in constant evaluation (e.g., if it only needs to work with the parameter by reference), it opts out in the usual way:

```rust
const fn ref_map<T, U, F: Fn(&T) -> U>(v: &Option<T>, f: &F) -> Option<U>
where
    // `T` is never dropped here; don't constrain const-context callers.
    T: do() Destruct,
{ ... }
```

The place you write it routinely is a returned `impl Trait`.  The automatic bound covers the function's parameters, which are demands on its callers — never its return type, which is a promise to them.  When const callers should be able to drop the value you return, promise that by adding `+ Destruct`:

```rust
const fn seed_hasher() -> impl Hash + Destruct {
    Fnv(0xcbf29ce484222325)
}

const SEED: u64 = {
    let h = seed_hasher();
    h.hash()
    // `h` is dropped here, which the bound allows.
};
```

The `Hash` and `Destruct` bounds elide to `do(~const)`.

### Traits that aren't const

What happens if a bound inside a `const fn` names a trait that never opted in?

```rust
trait Plugin { type Output; } // Not a const trait.

const fn run<T: Plugin>() { ... } // ERROR: `Plugin` is not a const trait.
```

This is an error.  There are two ways to fix it: mark the trait `const` (if you own it and its default function bodies can keep the promise) or write `T: do() Plugin` (if you only need its associated items).

### What you write in existing editions

For `const trait` and `const impl`, nothing differs in existing editions.

For `const` free functions and `const` functions in non-`const` inherent impls, until the next edition, unannotated bounds keep their plain meanings, and no `Destruct` bound is implied.  Write `~const` and `~const Destruct` explicitly where needed; the explicit syntax is available in every edition.

By desugaring, here is what a bare trait bound means in each position and edition:

| `Tr` bound in `..` | Existing editions | Next edition |
| --- | --- | --- |
| `const trait Tr2: ..` | `do(~const) Tr` | `do(~const) Tr` |
| `const trait Tr2 where ..` | `do(~const) Tr` | `do(~const) Tr` |
| `const trait Tr2 { type Ty: .. }` | `do(~const) Tr` | `do(~const) Tr` |
| `const trait Tr2 { fn f<..>() }` | `do(~const) Tr` | `do(~const) Tr` |
| `const trait Tr2 { fn f() -> impl .. }` | `do(~const) Tr` | `do(~const) Tr` |
| `const impl S where ..` | `do(~const) Tr` | `do(~const) Tr` |
| `const impl S { fn f<..>() {} }` | `do(~const) Tr` | `do(~const) Tr` |
| `const impl S { fn f() -> impl .. }` | `do(~const) Tr` | `do(~const) Tr` |
| `const fn f<..>() {}` | `do() Tr` | `do(~const) Tr` |
| `const fn f() -> impl ..` | `do() Tr` | `do(~const) Tr` |

### Migrating

When you run `cargo fix --edition`, bounds on your `const fn`s are rewritten to mean what they meant before.  Bounds on and within `const trait` and `const impl` are left alone as they mean the same thing in all editions.  The rules are:

- Plain bounds become `T: do() Tr`, except for sizedness, `Copy`, and auto trait bounds.
- Where required to preserve meaning, generic parameters of `const fn`s gain `+ do() Destruct`.  (The tool may skip the insertion where the tightening is provably unobservable.)
- Explicit `~const` bounds are left alone.  A machine-applicable lint will offer to tidy these up in the next edition.

Then you make a review pass over the inserted `do()`s.  Most of these should eventually be deleted (conditional constness is usually what you want), but each deletion affecting a function input asks more of const-context callers (and each deletion affecting a function output promises more), so delete with care on public functions.

Since trait bounds on `const` functions were of little use before this RFC, you likely won't have many of these to review.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### The bound grammar and the `do(..)` family

`TraitBound` gains an optional prefix modifier, available in all editions:

```text
TraitBound ->
      ForLifetimes? BoundModifier? TypePath
    | `?` TypePath
    | `(` TraitBound `)`

BoundModifier ->
      `do` `(` EffectModifierList? `)`
    | `~` `const` // Sugar for `do(~const)`.
    | `const`     // Sugar for `do(const)`.

EffectModifierList -> EffectModifier (`,` EffectModifier)* `,`?

EffectModifier -> `~`? `const`
```

The `for<..>` binder precedes the modifier; `?`-polarity and constness modifiers are mutually exclusive on a bound.

`do(~const)` and its `~const` sugar are permitted semantically only inside `const`-marked items: `const fn`s, `const trait` declarations, and `const impl`s (trait and inherent).  Associated items are inside them for this purpose: they carry no `const` of their own, and associated functions inherit the enclosing item's marking.  Items nested in function *bodies* are not: a helper `fn` declared inside a `const fn`'s body is its own, unmarked item.

`do(..)` is a set-replacement operator.  The effect modifiers on the bound are exactly those listed, replacing whatever the context would otherwise supply.  The recognized modifiers are `~const` (conditionally const) and `const` (always-const); `do()`, the empty set, is a plain bound.  Inside `const`-marked items, the elision rule below applies, and `do()` is the opt-out; everywhere else the context supplies nothing, and `do()` is a permitted redundancy.  Listing both `~const` and `const`, or listing a modifier twice, is an error.

The interior of `do(..)` parses as a comma-separated list of optionally-`~`-prefixed path-like names, resolved against a built-in registry that contains exactly `const`.  Nothing inside the parentheses needs to be a Rust keyword.

A bound modifier is permitted in generic parameter lists, `where` clauses, supertrait bounds, associated type item bounds (`const trait Tr { type Ty: ~const Ord; }`), associated-type bounds (`T: Iterator<Item: ~const Ord>`), higher-ranked bounds (`for<'a> T: ~const Visit<'a>`), argument-position `impl Trait` (which desugars to a generic parameter), and the bounds of return-position `impl Trait` (RPIT).

In RPIT bounds, the obligation falls on the hidden type at the definition site, and call sites assume the item bounds at matching strength (see below); the opaque carries no implied `Destruct` bound.  A modifier is not permitted in `dyn Trait` types.

Because `do` has been a reserved keyword since Rust 1.0 and `~` has no other meaning in the language, every form parses unambiguously in every position (including, for future work, in type position).

### The three strengths

Trait-bound constness is a *predicate* on the bound, not a generic parameter of the item (see [rust-lang/rust#131985]).  There is no "const version" and "runtime version" of a function or an impl, no constness token to pass around, and `<T as Tr>::Assoc` names the same type regardless of constness.  For a bound `T: Tr` with modifier *m*:

- **Plain** (*m* empty, `do()`): the ordinary trait obligation.  The bound's associated functions are not callable in a const context or when the surrounding item is invoked from one.
- **Conditionally const** (`~const`): in addition to the trait obligation, the item's body may call the bound's associated functions in const-eligible positions.  When the surrounding item is invoked from a const context with `T = X`, the obligation `X: const Tr` arises at that call site.  When invoked at runtime, no const obligation arises and any impl satisfies the bound.  The form is permitted only inside `const`-marked items.
- **Always-const** (`const`): when `T = X`, the obligation `X: const Tr` arises at every instantiation site, in every context.  The bound's associated functions are callable in the item's `const {}` blocks and other always-const positions as well as everywhere a conditionally const bound would allow.

An item's *const-eligible positions* are those whose code is const-checked as the item's own body.  A closure body belongs to the closure (a distinct function that does not inherit the enclosing item's constness), a nested item's body belongs to the nested item, and a `const {}` block is its own anonymous always-const item, as are the initializers of nested `const` and `static` items and the anonymous constants of const generic arguments and array lengths.

A bound's license attaches to the item that declares the bound: it covers that item's const-eligible positions and, for a bound on a `const trait` or `const impl` header, those of the item's associated functions, whose bodies are checked under the header's bounds.

`X: const Tr` holds when `X` has a `const impl` of `Tr` or when the obligation is discharged automatically: vacuously for the marker classes, structurally for the sizedness traits and `Destruct` (all below).  Always-const obligations also arise, independent of bounds, wherever code is unconditionally const-evaluated: `const` and `static` initializers, `const {}` blocks, const generic arguments, array lengths, and enum discriminants.

A const-context *call* imposes the const obligations of the item's conditionally const bounds at the call site; a runtime call imposes none.  Taking an associated function as a value (`let f = <X as Tr>::func;`) in a const context requires only the trait obligation: the const obligation attaches to calling, not to naming.

The item bounds of return-position `impl Trait` work similarly.  At the definition, the hidden type must satisfy the opaque's item bounds at their declared strength, proved under the function's own bounds: a conditionally const item bound discharges against the function's own conditionally const bounds.  At a call site, the item bounds enter the caller's environment at matching strength: a const-context call, having imposed the function's const obligations, may assume them at const strength; a runtime call assumes them plain.  For return-position `impl Trait` in a trait, the carrier is an anonymous associated type: the trait declares the item bounds, a `const impl` proves them for its own hidden type, and a plain impl proves only the plain trait obligations.

[rust-lang/rust#131985]: https://github.com/rust-lang/rust/pull/131985

### `const trait` declarations

`const trait Tr { .. }` declares a trait that admits const impls.  Its default function bodies are const-checked, under the trait's own conditionally const bounds and supertraits.  Making an existing trait `const` is a nonbreaking change; removing `const` is breaking.

Supertrait constness follows the same three-strength scheme as any bound: in `const trait Sub: Super {}`, a const impl of `Sub` must be accompanied by a const impl of `Super`, and `T: const Sub` gives const access to `Super`'s associated functions.  A plain (`do()`) supertrait on a const trait carries no const relationship.

Const traits remain ordinary traits in every other respect.

The associated functions of a const trait are written as plain `fn`.

### `const impl`

Writing `const impl Tr for X { .. }` makes `X: const Tr` hold and requires that `Tr` is a const trait and that every function body is a const body (under the impl's own conditionally const bounds).  A generic impl's discharge is conditional on its own bounds: under `const impl<T: ~const Tr> Tr2 for W<T>`, `W<X>: const Tr2` holds when `X: const Tr` does.

`const impl` also comes in *inherent* form, `const impl<T> W<T> { fn f() {} }`.  Every `fn` in such a block is a `const fn` (writing `const fn` inside is redundant and rejected); the header's bounds are treated as `~const` by default.

The header's generic parameters receive no implied `Destruct` bound (see below).

In the item grammar, each of the three forms takes a single optional `const` prefix, ahead of every other qualifier —

```text
Trait -> `const`? `unsafe`? `trait` IDENTIFIER ...

InherentImpl -> `const`? `impl` GenericParams? Type ...

TraitImpl -> `const`? `unsafe`? `impl` GenericParams? `!`? TypePath `for` Type ...
```

— with the rest of each rule unchanged (the unstable `auto` keeps its place after `unsafe`).  One form the grammar admits is rejected semantically: `const` on a *never impl* (also known as a negative impl), `const impl !Send for X`.  These unstable impls declare that no implementation will ever exist.  `const` would be an empty promise here.

### Conditional and always-const bounds require const traits

`T: ~const Tr` and `T: const Tr` are errors unless `Tr` is a const trait.

### Vacuous constness

For three classes of trait, the const obligation needs no `const impl`.  For these traits, `X: const Tr` holds exactly when `X: Tr` holds and `X` satisfies the const obligations of `Tr`'s own supertrait bounds:

1. auto traits (`Send`, `Sync`, `Unpin`, `Freeze`, `UnwindSafe`, ...).
2. `#[marker]` traits ([RFC 1268]).
3. `Copy` (keyed on the `copy` lang item, not on the name).

Such a trait still opts in by declaring itself `const trait`; the rule removes the need for const impls.  This RFC commits `Copy` to never gaining associated functions.

The standard library will declare `Copy` as:

```rust
const trait Copy: do() Clone {}
```

That `do()` is required.  Copying calls no associated functions, so const-copyability should not require const-cloneability.

[RFC 1268]: https://github.com/rust-lang/rfcs/pull/1268

### Sizedness bounds

The standard library declares the sizedness traits (`Sized` and the unstable `MetaSized` and `PointeeSized`) with `const trait`.  A bare sizedness bound inside a `const`-marked item elides to `do(~const)` (like any other bound), and the implied `Sized` bounds (on generic parameters, on associated types, and on the opaque types of return-position `impl Trait`) mean `do(~const) Sized` there too, exactly as if they had been written out.

For these traits, `X: const Sized` holds whenever `X: Sized` holds.  Every type that has a size today has one the compiler can compute at compile time.  Migrations do not add `do()` to sizedness bounds.

The rule is separate from the vacuous-constness rule above because const-sizedness is not vacuous: under the sized-hierarchy direction ([RFC 3729]), a scalable vector is `Sized` with a size known only at run time.

### The elision rule

Every trait bound written without an explicit modifier lexically inside a `const`-marked item means `do(~const)`.  For `const trait` declarations and `const impl` blocks, headers and associated items alike, this holds in every edition.  For `const fn`, it holds beginning with the next edition; in existing editions, an unmodified bound on a `const fn` stays plain.  The elision is a desugaring into the explicit forms.

The rule is syntactic:

- **In scope**:
  - Bounds in the generic-parameter lists and `where` clauses of `const fn`s (in the next edition).
  - Supertrait bounds of `const trait`s.
  - Bounds in the headers of `const impl`s (trait and inherent).
  - Bounds of associated functions of a `const trait` or `const impl`.
  - Item bounds of associated types of a `const trait` or `const impl`.
  - When other bounds are elided on an item, bounds of return-position `impl Trait`, the bounds of argument-position `impl Trait`, associated-type bounds, and higher-ranked bounds.
- **Out of scope**:
  - *Headers of non-`const` impls*, even when the impl contains `const fn`s: in `impl<T: Tr> W<T> { const fn f() {} }`, the header bound stays plain.
  - *Items nested inside function bodies*: a nested item is its own, unmarked item (per the position rules above), and its bounds take their own defaults.
  - *Type positions*: `dyn Tr` and `fn()` pointer types.
  - *`?`-relaxed bounds*: a relaxation is not a trait obligation and takes no modifier.
- **The opt-out**: an explicit modifier always wins; in particular `do()` produces a plain bound.  Elision applies only to bounds with no modifier.

During macro expansion, inside `const trait` and `const impl`, the elision is unconditional: the enclosing item's kind decides, whatever the edition of a bound's tokens.  For `const fn` bounds, edition hygiene governs: a bound's default follows the edition of the tokens that name the trait, so a bound a macro spells out follows the macro's edition, and a trait path interpolated into a macro-written bound keeps the edition of the code that wrote it.

[RFC 3729]: https://github.com/rust-lang/rfcs/pull/3729

### `Destruct`

`Destruct` is the const-droppability marker.  `X: Destruct` holds for every type.  `X: const Destruct` holds when `X`'s drop glue can run in constant evaluation: `X` has no drop glue; or `X`'s `Drop` impl is a `const impl` and each of `X`'s components is const `Destruct`.  (`Drop` itself becomes a `const trait` so that `const impl Drop` is writable.)  Dropping a value of a generic type in const-checked code requires the corresponding `Destruct` obligation.

`Destruct` is identified by its lang item and not by name or shape.  `Destruct` permits no user impls.

`X: Copy` implies `X: const Destruct`.

Each generic type parameter of an associated function in a `const trait` or `const impl` (and, in the next edition, of any `const fn`) carries an implied `do(~const) Destruct` bound (including the anonymous parameters that argument-position `impl Trait` desugars to).  The implication covers the function's *own* parameters only.  An explicit `Destruct` bound on the parameter replaces the implication: `T: do() Destruct` declines it; `T: const Destruct` strengthens it.

Conditionally const `Destruct` is not implied on the parameters of `const trait` and `const impl` headers (trait and inherent; `Self` counts as one of the trait's parameters), on associated types, or on the opaque type of a return-position `impl Trait` (in or out of a trait definition).

### Migration

The edition migration rewrites each unmodified trait bound `T: Tr` inside a `const fn` to `T: do() Tr` (the bounds of return-position `impl Trait` included), *except* where the new default cannot change observable meaning: `?`-relaxed bounds (no modifier is grammatical there), bounds naming the sizedness family (their const obligations are discharged for every current type), and bounds naming vacuous-class traits that are *declared `const`* and whose discharge reduces to the trait obligation (e.g., `T: Copy`, `T: Send`).

Each generic type parameter of a `const fn` also gains `+ do() Destruct` where the tool cannot prove the implied bound unobservable.

**The migration lint.**  In the old edition, a machine-applicable lint in the edition-compatibility group marks every to-be-rewritten bound and parameter inside `const fn`s with the rewrite that migration will apply.

### Lints

Explicit `~const` / `do(~const)` is redundant wherever a default supplies the same thing, as are explicit conditional bounds on const-declared vacuous-class traits and on the sizedness traits.  A machine-applicable lint offers to remove these.

Converting a block of `const fn`s in a non-`const` inherent impl into a `const impl`, in existing editions, strengthens any contentful bare bound (Drawbacks weighs this).  To mitigate this, a machine-applicable lint offers that conversion with the migration rewrite above applied, so that the conversion is meaning-preserving.

## Drawbacks
[drawbacks]: #drawbacks

**Some crate authors skip edition migrations.**  The automated edition migration preserves semantics.  But if a crate bumps the edition without running it, some bounds silently strengthen.  That path has never been supported, and there have always been hazards in taking it.  Here, the hazard will be caught at compile time (even if downstream), which is a smaller hazard than other edition items we've shipped successfully.

**Context dependence.**  Bounds within `const`-marked items will be `~const` by default while other bounds won't be.  Generally this will be what people want, but it will be another rule to learn.  Survey data suggested people were split on their expectations.  The `?Sized` history suggests we need to approach this carefully.  But our experience with lifetime elision, implicit `Sized` bounds, and the RPIT lifetime capture rules suggests there is overall benefit to choosing good defaults, particularly when we can desugar to a fully explicit form.

**Intra-edition tension.**  In existing editions, for backward compatibility, only bounds in `const trait` and `const impl` will infer `~const`, meaning that `const fn` bounds will work differently.  We'll make this consistent in the next edition.  We earlier used this trick when RPIT in traits and in trait impls shipped in Rust 1.75 with the 2024 lifetime capture rules ([RFC 3498]), ahead of the 2024 edition (Rust 1.85), which applied the rules consistently.  We prefer *inter*-edition consistency over *intra*-edition consistency ([RFC 3085]).

**Converting `const fn`s into a `const impl` changes bounds.**  Converting `impl W { const fn f<T: Ord>(..) }` to `const impl W { fn f<T: Ord>(..) }` changes `T: Ord` to mean `T: ~const Ord + ~const Destruct`.  But the bounds that dominate today's `const fn`s (sizedness, `Copy`, `Send`) are the ones unaffected by this move.

**Hoisting a bound can change its meaning.**  In the next edition, moving `where T: Trait` from a `const fn` up to a surrounding non-`const` `impl` header removes the implied `~const`.  We expect that `const fn` within non-`const` inherent impls will become less common and that it will be more idiomatic to write a `const impl` and put the const functions within that.

**Elided return bounds are promises.**  Writing `-> impl Iterator` on a `const`-marked item promises a conditionally const iterator.  Unlike with bounds on input generics, this is in the less SemVer-safe direction.  We accept this for consistency with `~const` elision on the item bounds of associated types.

**Droppable returns take an explicit bound.**  `~const Destruct` is not implied on the item bounds of associated types or of return-position `impl Trait`, so `+ Destruct` will need to be written where const callers might drop the value.  Adding it later is backward compatible.

**Migration adds temporary noise.**  To preserve semantics, the edition migration rewrites plain bounds to `do() Tr` and adds `+ do() Destruct` to the generic parameters of `const fn`s.  Likely most authors will want to remove these and move toward conditionally const semantics.

**Implementation and maintenance burden.**  Opt-out-shaped features have historically grown subtle bugs.  The elision can be framed as a syntactic lowering to a common desugaring, but that's not necessarily how it would be implemented.  And implementing and testing the elision could lengthen the road to stabilization.

**Closures in `const`-marked items aren't const.**  Following the spirit of recent lang decisions (see [rust-lang/rust#157273]), one would expect closures within `const`-marked items to be const closures.  If we could do that in this RFC, closures within `const trait` and `const impl` would work the same way in all editions, and we'd migrate closures within `const fn` at the next edition.  This RFC doesn't do that, as a concession to lack of implementation readiness.

**Churn.**  Nearly all of the standard library's 474 existing conditional annotations will become redundant, and a machine-applicable lint will offer to remove them.  The ecosystem will constify mostly by adding `const` to items.

[rust-lang/rust#157273]: https://github.com/rust-lang/rust/pull/157273

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Should conditionally const stay explicit forever?

*TODO*.

### Imply only `Destruct`

*TODO*.

### Why `const trait` and `const impl` are edition-independent

*TODO*.

### Marker traits

*TODO*.

### Would `async` be next?

*TODO*.

### Why `~const` and `do(..)`

*TODO*.

### Why `do()` for the opt-out

*TODO*.

### Design of `Destruct` inference

*TODO*.

#### Why `Self` and the header generics stay explicit

*TODO*.

### Why not per-function constness, automatic const, and `~const fn`

*TODO*.

[RFC 3085]: https://github.com/rust-lang/rfcs/pull/3085
[RFC 3498]: https://github.com/rust-lang/rfcs/pull/3498
[RFC 3617]: https://github.com/rust-lang/rfcs/pull/3617
[RFC 3654]: https://github.com/rust-lang/rfcs/pull/3654
[rust-lang/rust#144207]: https://github.com/rust-lang/rust/issues/144207
[rust-lang/rust#73255]: https://github.com/rust-lang/rust/issues/73255

### The counterexamples, answered

["Const Trait Counterexamples"][counterexamples] ([GitHub source][counterexamples-gh]) argues against elision.  In this section, we restate each problem the post suggests (keeping its identifiers), then answer it.

> **1A — edition migration.**  Redefining bare bounds inside `const fn` is a breaking change, so it must be staged across editions, and crates that bump the edition without running the migration would silently have bounds tightened.

On an edition boundary, we need to migrate code to the intersection between two editions.  The explicit syntax is stable in all editions.  At the edition boundary, we migrate code to the explicit form.  This allows us to do the migration over one edition rather than over multiple, as the counterexample assumed.

Since the legacy behavior is contained to `const fn` in this RFC, that's all we need to migrate.  While they exist, trait bounds on `const fn` are relatively rare (making them far more useful is the motivation of this RFC).  Bounds on `const trait` and `const impl` will already have the new behavior.

While some people do bump the edition without running migration lints, this has never been supported and there have always been hazards in doing that.  Here, the hazard will be caught at compile time (even if downstream), which is a smaller hazard than other edition items we've shipped successfully.

> **1B — bounds over non-`const` traits.**  A bare bound naming a non-`const` trait can't elide and can't stay plain without making it breaking to make the trait `const` and its functions' bounds `~const`, so it must be an error, and that would be confusing.

Naming a non-`const` trait in a `~const`-elided bound must indeed be an error.  It would be an error if written explicitly, and this RFC treats elided bounds as desugaring to the explicit form.

The claim of confusion relates to the 1C and 1D items, which we'll address below.

> **1C — impl-block confusion.**  Non-`const` impl blocks holding `const fn`s are confusing either way: eliding in header bounds strengthens too much, while not doing so makes the rules complicated and inconsistent.

This RFC does not elide in the headers of non-`const` impl blocks.  The rule we choose is syntactic and scoped to the item.  Inside a `const`-marked item, bare bounds default to `~const`.

The survey findings suggest that, whatever the rules, people may be confused when mixing a non-`const` inherent impl with `const fn`.

We expect that `const fn` within non-`const` inherent impls will become less common.  It will be more idiomatic to write a `const impl` and put the const functions within that.  This will help resolve any remaining confusion.

> **1D — the usage path.**  Compiler errors prompt an opt-in `~const` where it's needed, but nothing prompts the opt-out, so signatures may demand too much.

This is true, but demanding too much is the safer direction in a SemVer sense.  We want it to be explicit when library authors are making guarantees.

Guaranteeing that a const function will *never* need to call a trait's associated functions is the stronger guarantee, so we ask authors to write that explicitly.  An author can add this guarantee later without breaking any downstream code.

> **1E — virality toward types.**  `dyn Trait` and `fn()` types are valid in `const fn` signatures today; an elision presses toward either spreading onto those types or stopping at an inconsistent place.

This RFC declines to extend elision to `dyn Trait` or `fn()` types.  We think it makes sense, even long term, for `~const` on these to be stated explicitly.

We extend elision only to the universal and existential bounds of functions, impls, and traits, i.e., to bounds on generic parameters, associated type item bounds, and return-position `impl Trait` item bounds.

Given the syntactic nature of the elision rule, it would be odd not to elide associated type item bounds.  And there's a deep connection between associated types and return-position `impl Trait` — for example, within trait definitions, these desugar directly to anonymous associated types.  So we should preserve parity, and this is the more useful default.

But `dyn Trait` and `fn()` types are different.  These are types, not bounds on the `const`-marked item.  And we already accept differences between these and RPIT.  The lifetime capture rules ([RFC 3498]), for example, apply to `impl Trait` but not to `dyn Trait`.

Conveniently, this matches how the type system thinks about things.  Conditionally const bounds on RPIT are already supported in the implementation, while `~const` on `fn()` and `dyn Trait` would require extensive type-system work.

[counterexamples]: https://dbeef.dev/const-trait-counterexamples/
[counterexamples-gh]: https://github.com/fee1-dead/website/blob/1fdbf1f5d287d52699ce02b1ec7f987e0b5c3f09/content/const-trait-counterexamples.md

## Prior art
[prior-art]: #prior-art

**[RFC 3762]**.  We adopt the core semantic model of RFC 3762, including the separate `Destruct` trait.

**[RFC 2632]** (2019).  RFC 2632 proposed implicit conditional constness for bounds in `const fn` with `?const` as the opt-out.  It lacked a separate `Destruct` trait, instead overloading `Drop` for this.

**The lang experiment**.  [RFC 2632] was closed in favor of a nightly lang experiment, which has been ongoing.

**Implicit `Sized` bounds**.  While `?Sized` is confusing, Rust would be worse if all `Sized` bounds had to be written explicitly.  This RFC leans on that lesson.

**Lifetime elision**.  Lifetime elision shows that people can learn a straightforward elision rule.  It shows the value of good defaults.  And it shows that we can offer good defaults while desugaring to an explicit form, preserving expressiveness.

**RPIT lifetime capture rules**.  In RPIT (and RPITIT), we capture all in-scope generic parameters.  As with lifetime elision, we offer a fully explicit form (`use<..>` bounds).  As with implied `Sized` bounds, we offer a way to opt out (`use<>`).  Before edition 2024, the rules worked differently, and we migrated to the current rules over an edition.  As this RFC does, we leaned on having a fully explicit syntax to make that migration possible.

**C++ `constexpr`**.  While not directly comparable to Rust (because of the lack of bounds), the pervasiveness of this annotation highlights the risk of noisy defaults.

**Zig `comptime`**.  Zig's `comptime` code is checked at instantiation, with no declared bounds.

**User expectations**.  The [2026 survey][const-traits-survey] found respondents split on what an unannotated bound in a `const fn` should mean, with maintainers preferring conditional-by-default more strongly than other respondents.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- The placement of the `Destruct` trait in the hierarchy of the standard library is delegated to the libs-api team.

## Future possibilities
[future-possibilities]: #future-possibilities

- **Const trait objects and const function pointers.**  `dyn ~const Tr`, `~const fn()`, and always-const variants of each are needed.  When done, elision will not apply to these.
- **Const closures.**  `const || ..` closures are needed and left to future work.
- **More effect-like qualifiers.**  This RFC is forward compatible with possible future qualifiers such as `nopanic`.
- **Item-level `do(..)`.**  If we later allow `do(..) fn`, we could support future qualifiers without making them keywords.
- **Comptime.**  This RFC is forward compatible with ongoing work on functions that may *only* be called at compile time.
- **Always-const associated functions.**  We could later offer a way to mark associated functions in a const trait as being always-const so that they can be called in a const context within the trait definition itself and in other places where only a non-`const` bound is available.
- **Declaring `Copy` with `#[marker]`.**  Today `Copy` isn't declared with `#[marker]`.  The rules in this RFC are compatible with later doing that.
- **Generic const items.**  If we later support generics on `const` items, we'll want to give their bounds an always-const default.
- **Const derives.**  We may later want something like `#[derive(const Trait)]`.

## Acknowledgments

Thanks to both oli-obk and fee1-dead for pushing forward this work over many years.  Thanks to oli-obk for [RFC 2632] and [RFC 3762].  Thanks to fee1-dead for writing ["Const Trait Counterexamples"][counterexamples], which sharpened the design and the arguments in this RFC.  Thanks to compiler-errors, who built the underlying predicate representation ([rust-lang/rust#131985]).  And thanks to many others for useful feedback and suggestions.

All errors remain those of the author alone.

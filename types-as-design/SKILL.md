---
name: types-as-design
description: Push domain invariants into the type system so invalid states become unrepresentable. Use when modeling domain state, replacing boolean/optional-field piles with discriminated unions, or when `if` checks on field combinations are multiplying across a codebase.
---

# Types as Design

Every `if` that checks a combination of fields is a type that doesn't exist yet. Runtime checks are the tax you pay for a type that can't express your domain — and the tax compounds: the same combination gets re-checked at every use site, and a missed site becomes a bug.

The fix is not more checks. It is a better type. Make invalid states unrepresentable: if two flags can't both be true, they are one enum. If a field only exists in one state, it belongs on that state's variant, not as optional on the shared shape. A check you delete because the compiler forbids the case is a bug that can never ship.

## The decision rule

Push an invariant into the type when **both** hold:

1. **It is a real domain rule** — the business would call it an error, not a preference. "A closed order has no editable cart" is a rule. "We usually pass the name too" is not.
2. **The type stays simpler than the checks it replaces** — a reader can look at the type and state the rule in one sentence.

The gate decides whether a rule belongs in the type at all. Each technique's "Reach for this when" line then tells you which construct fits once you are through it — and where a technique carries no domain rule to weigh, the gate has nothing to decide and the trigger stands alone.

Do NOT type-golf: a type that needs a comment to explain is worse than a runtime check with a clear name. If expressing the invariant requires conditional-type gymnastics, the invariant belongs in a function with tests.

## Technique catalog

### 1. Discriminated unions over boolean/optional piles

```ts
// Before — most combinations are meaningless, every consumer re-derives the real state
type Fetch = { loading?: boolean; data?: Data; error?: Error };

// After — exactly the states that exist
type Fetch =
  | { kind: 'idle' }
  | { kind: 'loading' }
  | { kind: 'ok'; data: Data }
  | { kind: 'err'; error: Error };
```

Reach for this when a shape has two or more fields that are mutually exclusive or co-required. The tell: callers write `if (!loading && data)` to reconstruct a state the type should have named. Each variant carries only the data valid in that state — `data` exists on `'ok'`, so no consumer can touch it while loading.

### 2. Exhaustiveness with `never`

```ts
function render(f: Fetch) {
  switch (f.kind) {
    case 'idle': return idleView();
    case 'loading': return spinner();
    case 'ok': return show(f.data);
    case 'err': return showError(f.error);
    default: const _: never = f; throw new Error(`unhandled: ${_}`);
  }
}
```

Reach for this at every switch over a union you control. Adding a variant now breaks the build at every unhandled site — the compiler becomes your TODO list. Without the `never` assertion, new variants silently fall through and you get a runtime "should never happen" you will, in fact, hit.

### 3. Branded types for primitive obsession

```ts
type UserId = string & { readonly __brand: 'UserId' };
type OrderId = string & { readonly __brand: 'OrderId' };

function charge(userId: UserId, orderId: OrderId) { /* ... */ }
```

Reach for this when two or more primitives flow through the same code and swapping them is a live bug — `charge(orderId, userId)` compiles with plain strings and charges the wrong thing. Brand at parse boundaries (schema parsing, deserialization), not by casting mid-code. An `as UserId` deep in the interior means your boundary let a raw string in, so the fix belongs at that boundary rather than at the call site. The Rust/Go idiom is the newtype wrapper (`struct UserId(String)`); in TS the brand is zero-cost because it's erased at runtime.

### 4. Parse, don't validate

```ts
// Before — returns boolean, proves nothing; every caller re-checks or trusts blindly
function isValidConfig(x: unknown): boolean { ... }

// After — returns a refined type; the proof travels with the value
function parseConfig(x: unknown):
  | { ok: true; config: Config }
  | { ok: false; error: ParseError } { ... }
```

Reach for this at every point where untrusted data enters. Validation answers a question and then forgets the answer; parsing changes the type, so every downstream function can demand `Config` and never re-check. Parse at the boundary, pass the refined type inward.

### 5. Option/Result over null-and-throw

Reach for this for **expected** failures — a lookup that misses, a parse that can fail, a payment that can decline. In TS, make failure a union member (`{ ok: true, value } | { ok: false, error }`) rather than throwing; exceptions are for states you believe are impossible. The tell for the wrong choice: a `try/catch` whose `catch` does real business logic — that failure was expected and should have been in the type. In Rust this is `Option`/`Result` natively; in Go it's the `(T, error)` pair — either way, the caller must acknowledge failure to get the value.

### 6. Readonly and const assertions

```ts
const ROUTES = ['home', 'settings', 'admin'] as const;
type Route = (typeof ROUTES)[number];

function transfer(state: Readonly<Account>) { ... }
```

Reach for this when mutation is the default and shouldn't be. `as const` turns a loose array into an exact set of literals. `Readonly<T>` on a parameter makes mutation a decision the caller opts into, not an accident a callee commits. If a function needs to mutate, its signature should say so by taking the mutable type.

## Where the boundary is

Types must absorb untrusted input at system edges — API payloads, env vars, files, form data. Parse there, then keep the interior of the codebase in refined types with **no re-validation**. If you find yourself re-checking an invariant deep in the interior, the boundary type failed: either the parse was too loose, or the refined type never made it inward. Fix the boundary, don't add another interior check — interior checks are duplicates that drift out of sync with each other.

## Anti-patterns

- **Boolean explosion** — N booleans imply 2^N states, most meaningless. The tell: a comment or wiki page explaining which flag combinations are legal. Replace with one discriminated union (technique 1).
- **Optional-field soup** — every field `?`, every use site guessing which subset is present. The tell: `obj.field?.nested?.thing` chains and `if (a && b && c)` guards scattered across callers. The fields co-occur in states; name the states.
- **Stringly typed** — status as a free `string` where the union already exists, just living in JSDoc or a teammate's head. The tell: string comparisons against literals (`if (status === 'active')`) with no type-level guarantee. Write the union down.
- **Cast laundering** — `as` used to silence the compiler instead of fixing the model. An `as` is an admission the types don't express reality. The tell: a type assertion anywhere except a parse boundary, or `as unknown as X` anywhere at all. Fix the model; a cast you keep anyway carries its justification in a comment beside it, so the next reader argues with a reason instead of a mystery. `as const` is exempt — it narrows a literal without asserting anything the compiler can't already see.
- **Type golf** — conditional-type wizardry that saves three lines and costs every reader ten minutes. The tell: the type definition is longer than the function it describes, or a teammate asks what it does. If it needs a comment, a plain type plus a runtime check was the better design.

## Completion checklist

- [ ] No meaningless boolean combinations remain — every pair of flags that can't co-exist is one enum or union
- [ ] State is carried on the variant that owns it — no `?` field that is only present in one state
- [ ] Untrusted input is parsed once at the boundary — no re-validation in the interior
- [ ] No type assertion outside a parse boundary — any kept one carries its justification beside it, `as const` excepted, and never `as unknown as`
- [ ] Exhaustiveness checks (`never` default) at every switch over a variant union you control
- [ ] Expected failures are union members or `Result`-style returns — `catch` blocks contain only crash handling

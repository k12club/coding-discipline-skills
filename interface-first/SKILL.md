---
name: interface-first
description: Design a module's interface before writing any implementation. Use when building a new module, class, or public API, when a refactor will change a module's boundaries, or when you catch yourself writing function bodies before signatures.
---

# Interface-First Design

The failure mode this prevents: code written before its interface is designed ends up shaped like the implementation, and every caller pays for it forever. Implementation details leak into signatures, callers acquire knowledge they shouldn't have, and the cost compounds with every new call site. The interface is the cheapest place to be wrong and the most expensive place to be wrong later — so design it first, on purpose, while changing it costs nothing.

**No implementation until the interface exists on screen.** If you catch yourself writing a function body while its module's interface is still in your head, stop — bodies written in that state exist to justify an interface nobody chose.

When exploring the codebase, read `CONTEXT.md` (if it exists) so interface vocabulary matches the project's domain language, and check ADRs in the area you're touching.

## The depth test

A module should be **deep**: a small, simple interface with substantial functionality behind it. Depth is the ratio of "what callers must learn" to "what the module does for them." A deep module is like the filesystem — five calls, everything hidden. A shallow module is a wrapper that costs more to learn than it saves.

Tells of a shallow module:

- **Interface complexity ≈ implementation complexity.** Reading the signature list teaches you roughly everything the module knows. There is nothing behind the curtain.
- **Arguments pass straight through.** The method forwards its parameters to another method unchanged; the caller could have called the target directly.
- **More getters than logic.** The class is a data bag with ceremony. The logic that should live inside it lives in its callers instead.
- **Every method needs the caller to know something internal** — an ordering, a flag combination, an ID format, a cleanup call afterward.

If the module you're about to build scores shallow on these tells, the interface is the problem, not the implementation. Fix it in Phase 2.

## Phase 1 — Write the interface first

Literally write it: type signatures, class skeleton, function signatures with doc comments — in the real file, before any body. Not in a scratch note, not in your head. On screen, in the language's actual syntax, where it can be read, reviewed, and stress-tested.

```ts
/** Manages the on-disk cache for compiled templates.
 *  Callers never see file paths, eviction order, or the lock file. */
interface TemplateCache {
  /** Returns the compiled template, building it on first access. */
  get(templateName: string): CompiledTemplate;
  /** Drops all cached entries. Safe to call at any time. */
  clear(): void;
}
```

The doc comments are not decoration — they are where you state what the module hides. If you cannot write the comment without mentioning an internal detail ("pass `null` to skip validation", "call `flush()` before reading"), that detail is about to leak. Redesign the signature until the comment stays clean.

Do not proceed to Phase 2 until the skeleton compiles with empty bodies.

## Phase 2 — Generate 2–3 radically different candidates

One interface is a first guess; three is a design. Generate 2–3 candidates that are **radically different**, not variations of one idea. Radically different means they disagree about at least one of:

- **The seam** — where the module's boundary is drawn (one object vs. a pipeline of stages; push vs. pull; sync vs. async).
- **Data ownership** — who holds the state (the module keeps it vs. callers pass a handle vs. stateless pure functions).
- **Error model** — exceptions vs. result values vs. errors defined out of existence entirely.

If your three candidates share the same shape with different names, you have one candidate. Keep pushing until the differences are structural.

For each candidate, write down three things:

1. **What callers write** — a real 3-line usage snippet, in the project's language. This is the payload; it makes "simple" concrete and falsifiable.
2. **What it hides** — the implementation decisions callers never learn about.
3. **What it leaks** — what callers must know, carry, or handle that they cannot affect. Leaks are permanent; name them honestly.

Present the candidates to the user with a recommendation and let them pick. This is a cheap checkpoint before expensive implementation — an interface is a paragraph; rewriting one after implementation is a refactor. Don't block if the user is AFK: proceed with your recommendation, and mark it as chosen-by-default.

## Phase 3 — Stress-test the chosen interface

Before any implementation, walk through the 2–3 **hardest real use cases** end-to-end, writing only caller code against the interface. Hardest means: the use case with the most state, the one with partial failure, the one the current callers actually do that the happy path ignores. Not hypothetical use cases — real ones, pulled from existing call sites or the requirements.

Write the caller code out, in full, in the real syntax. Then interrogate it:

- **What information does the caller need that it shouldn't?** If caller code builds an internal ID, orders its calls, or branches on a module-internal condition, the abstraction is leaking. Pull that knowledge inside.
- **What does the interface expose that callers must handle but can't affect?** A callback they must supply but can't influence, a state they must track but can't change, a return value they must check but can't act on. That is a leak, not a feature.
- **Can errors be defined out of existence?** Before adding an error case, ask whether it can simply never occur. Make `clear()` a no-op on an empty cache instead of throwing. Make "unset" a legal, meaningful state instead of an invalid one. Every error you define out of existence is exception-handling code deleted from every caller, forever.

**If the caller code is awkward, the interface is wrong — fix the interface, don't document around it.** A doc comment explaining a workaround is a confession that you shipped the awkwardness. Go back to Phase 2 with what the caller code taught you.

If you catch yourself thinking "callers will just have to remember to…", stop — that sentence always precedes a leak.

## Phase 4 — Implement

Only now write bodies. The interface is a contract; the implementation's job is to disappear behind it.

Hard rule: **if implementation reveals the interface must change, go back to Phase 2 explicitly** — regenerate candidates with the new constraint, re-present if the shape changes materially, re-run Phase 3's caller walkthroughs. Do not silently mutate the interface mid-implementation. Silent mutation is how "designed" interfaces decay into implementation-shaped ones: each small concession looks harmless, and you wake up with getters and a `flush()`.

An interface change discovered during implementation is not failure — it means Phase 3's stress cases missed something, which is worth noting. But it re-enters the process at Phase 2, not at the diff.

## Anti-patterns

- **Pass-through method** — a method whose signature mirrors the one it calls, forwarding arguments unchanged. The tell: deleting the method and letting callers call the target directly loses nothing but a file. It adds an argument list but no abstraction. Either absorb real responsibility (defaults, validation, state) or delete it.
- **Over-specialized interface** — designed for the one caller that exists today, shaped exactly like that caller's current code. The tell: the signature contains vocabulary only that caller uses, and the second caller you imagine needs every parameter changed. Somewhat-general beats perfectly-fit-to-one-caller; design for the use case category, not the instance.
- **Temporal decomposition** — the interface mirrors the steps of the current implementation (`step1_load()`, `step2_parse()`, `step3_emit()`). The tell: callers must call methods in a fixed order, and reordering the internals would break every caller. The interface should expose *what* the module does, not the *sequence* in which today's implementation happens to do it.
- **Speculative generality** — options, config objects, and plugin hooks for futures nobody asked for. The tell: a parameter with exactly one value in every call site, or a flag whose other branch is untested. "Somewhat general" beats "perfectly general": general enough for today's real use cases, simple enough that the next one can refactor it. Generality you can't test is complexity you can't remove.

## Completion checklist

Before declaring the module done:

- [ ] Interface (signatures + doc comments) was written in the real file **before** any implementation body.
- [ ] 2–3 **radically different** candidate interfaces were presented to the user, each with a caller snippet, hides-list, and leaks-list, and one was chosen.
- [ ] The 2–3 hardest real use cases were walked through end-to-end as caller code, and the awkwardness found there was fixed in the interface — not documented around.
- [ ] Every error case was challenged: the ones that could be defined out of existence were, and the survivors are ones callers can actually act on.
- [ ] Any interface change discovered during implementation went back through Phase 2 explicitly, not through silent edits.
- [ ] Final depth check: the interface is smaller than the functionality behind it, and no caller needs to know an internal detail to use it correctly.

## Interplay

This skill pairs with a deep-modules vocabulary skill (e.g. `codebase-design`) if one is installed — load it for the shared language around seams, depth, and information hiding. This skill is self-contained: everything above works without it.

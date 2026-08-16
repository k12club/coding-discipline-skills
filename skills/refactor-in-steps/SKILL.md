---
name: refactor-in-steps
description: Restructuring discipline for refactors. Use when the user asks to refactor, restructure, clean up, or rename across a codebase — any change that should alter structure without altering behavior, especially ones spanning multiple files or callers.
---

# Refactor in Steps

A refactor that can't be done in steps is a refactor you don't understand yet. If you cannot name the sequence of individually-verifiable moves that gets you from here to there, you don't have a plan — you have a hope, and hope produces one giant broken diff.

Small steps are not caution for their own sake. They are how you keep the ability to **bisect** (which step broke it?), **revert** (undo one step, not the whole afternoon), and **review** (each diff readable in one sitting). A single 40-file diff has none of these; a sequence of ten 4-file diffs has all of them.

## Definitions — enforced

A **refactoring** changes structure without changing observable behavior. Same inputs, same outputs, same side effects — different shape.

A **behavior change** is anything a caller could notice: different output, different error, different timing contract, different default. It is a different commit.

Never mix the two in one step. If the work needs both, it is two steps — restructure first (behavior pinned by tests), then change behavior on the clean structure. The tell that you mixed them: the step's commit message needs the word **"and"**. "Extract interface *and* handle nulls" is two steps pretending to be one.

## Phase 1 — Characterization before movement

Before moving anything, tests must pin current behavior at the seams you're about to restructure. Not aspirational behavior — **what the code does today, including the quirks**. The weird null-coalescing, the implicit ordering, the error swallowed on Tuesdays: characterize all of it, because the refactor must preserve all of it.

If coverage at the target seams is missing, the **first step is writing characterization tests** — not touching code. These tests are allowed to look ugly and over-fit; their job is to freeze the present, not to be a spec. Mark them (`// characterization: pins pre-refactor behavior`) so a later cleanup can revisit them.

No characterization, no refactoring. If you catch yourself restructuring untested code "carefully", stop — careful is not a verification strategy. Do not proceed until the seams you're about to cut have tests that currently pass.

## Phase 2 — Plan the step sequence

Write the ordered list of steps **before touching code**, in the commit messages if possible. Each step must be:

1. **Small** — you can hold the whole diff in your head and describe it in one sentence without "and".
2. **Behavior-preserving** — characterization tests pass before and after, unmodified.
3. **Green-preserving** — the codebase builds and tests pass at the end of the step. No step ends broken expecting the next step to fix it.

Test the plan before executing it: for each step, ask "what test run proves this step worked?" If a planned step can't be made green-preserving — it requires a half-migrated state where nothing compiles — **the plan is wrong**. Split the step, or add a seam first (see Phase 3, pattern 5). Rewriting the plan now costs minutes; discovering it mid-diff costs the diff.

Show the sequence to the user before starting on anything larger than a few files. They know which step will collide with in-flight work.

## Phase 3 — The step patterns

Five moves cover nearly every restructure. Learn them as vocabulary; most steps are one of these.

### 1. Expand–contract

Add the new structure alongside the old, migrate callers one by one, delete the old last. Never rename-and-rewrite in one move — that produces a diff where every caller changed *and* the implementation changed, and review becomes trust-me.

```
// step 1 (expand): both exist, new one tested
function totalPrice(items) { ... }        // old, still called
function priceForItems(items) { ... }     // new, tested against old

// steps 2..n (migrate callers, one diff per batch)
- const t = totalPrice(cart);
+ const t = priceForItems(cart);

// final step (contract): delete old
```

### 2. Parallel change

Expand–contract across an interface boundary, for signature or API changes. Add the new signature, migrate callers, remove the old. Works across module and deploy boundaries — the old signature stays live until the last caller moves, so nothing is ever broken between steps.

```
function lookup(key) { ... }                  // old
function lookupEntry(key, { fallback }) { ... } // new, delegates

// callers migrate incrementally; old is deleted only when unused
```

### 3. Strangler

For replacing a whole subsystem: put a façade in front, route calls through it, move logic behind the façade one piece at a time. The old path dies when the last caller moves — and not before.

```
// façade delegates; move one branch at a time behind it
function route(req) {
  if (newPathHandles(req)) return newHandler(req);
  return legacyHandler(req); // shrinks every step
}
```

### 4. Move-then-modify

Relocate code **verbatim** in one step; adjust it in the next. The move diff shows pure cut-and-paste — reviewable in seconds. Moving and editing in one step makes the diff unreviewable: the reviewer must prove that 200 moved lines contain no hidden change, and they can't.

Rule: the move commit contains no logic edits. Fix imports, paths, and exports only. If the code needs changes to survive the move, extract a seam first so it doesn't.

### 5. Extract seam

When the code resists a step — every cut you try breaks ten things — stop cutting and first extract an interface or function so the next step has a clean place to slice. The seam extraction is itself a step: behavior-preserving, green, committed alone.

```
// before: processOrder does validation, pricing, persistence inline
// step: extract so the next step can replace pricing only
function processOrder(order) {
  validateOrder(order);
  const total = priceOrder(order);   // ← the seam
  persistOrder(order, total);
}
```

## Phase 4 — Execute one step at a time

The loop, per step:

1. Implement the step. Nothing else.
2. Run the tests that pin the seams you're touching.
3. Tests green → commit (or clearly mark the step boundary if commits aren't yours to make).
4. Next step.

If tests go red mid-step and you can't see the cause immediately, **revert the step — do not patch forward**. A red step whose fix you can't see means the step was too big; splitting it is the fix, and you can't split it from inside a broken state. Reverting costs one step. Patching forward compounds an unknown into the next step and forfeits bisectability — the entire point of the sequence.

Do not batch steps "because they're all small anyway". Smallness is judged per step, verified per step.

## Anti-patterns

- **Big-bang rewrite** — "while I'm in here" energy: the diff touches everything, no intermediate state ever compiled, review becomes trust-me. The tell: you can't point to a commit where the codebase was both restructured and green.
- **Mixed commit** — refactor + behavior change + formatting in one diff. The tell: the commit message needs "and", or the reviewer asks "wait, did this line change behavior?" and you have to think about it. Split it; formatting-only changes get their own commit or none.
- **Leap of faith** — a step so large it can only be verified by "the whole suite still passes, probably". The tell: you run the full suite because no targeted test could isolate what you changed. If the only verification available is global, the step was too big — revert and split.
- **Abandoned migration** — expand without contract: old and new structure coexist forever, and every future reader pays the ambiguity tax. The tell: the step plan has no deletion step at the end. Every expand–contract or parallel-change plan **must end with a deletion step**, scheduled, not aspirational.

## When to break the rules

Ceremony scales with blast radius: **callers × distance** of the change. A rename of a local variable with two references in one file needs no characterization tests and no step plan — just do it. A signature change with forty callers across three packages needs the full protocol. Judgment lives in the middle; when unsure, write the step list — if it takes more than a minute to write, you needed it.

## Completion checklist

- [ ] Step sequence written down before code was touched
- [ ] Characterization tests exist at every seam being restructured, and pass unmodified
- [ ] Every step individually green and individually committed (or boundary-marked)
- [ ] No step mixes structure change and behavior change (no "and" in any step message)
- [ ] Every move step is verbatim; modifications happened in separate steps
- [ ] Final step deletes the old structure — nothing abandoned in expand state

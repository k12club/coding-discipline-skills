---
name: read-before-write
description: Reconnaissance discipline before editing code. Use before any non-trivial edit to an existing codebase — adding a feature, fixing a bug, touching a file you haven't read end-to-end.
---

# Read Before Write

The diff you write should look like the codebase wrote it. Your default style is an average of the internet — the library you always reach for, the naming you default to, the error idiom you find tasteful. The project's style is specific, and it almost never matches your average. That gap is invisible to linters and compilers, and glaring to every human who reviews your diff. The failure is not a skill failure; it is a reconnaissance failure. You edited before you read.

This skill is the reconnaissance. Run Phases 1–4, in order, before the first keystroke of an edit; Phase 5 is the edit itself. Skip the protocol only when the edit is genuinely trivial (a typo, a one-line constant) — and be honest with yourself about what "trivial" means.

## Phase 1 — Locate the pattern

Before writing anything, answer one question: **has this project already solved this exact problem?**

Search before you write:

- Grep for the capability by name and by synonym — `retry`, `backoff`, `sleep`; `formatDate`, `toISO`, `parseDate`; `logger`, `log`, `warn`.
- Look for sibling modules doing the analogous thing. Adding a new API endpoint? Read an existing endpoint. Adding a new error type? Read the error module.
- Check for a shared utilities directory (`utils/`, `lib/`, `internal/`, `common/`) — that is where the second copy of your function is about to be born.

If an existing pattern solves 80% of your problem, **extend the pattern** — add a parameter, a variant, a case. Do not introduce a second way to do the same thing. Two ways to do one thing is a tax every future reader pays: they must learn both, decide which is canonical, and guess wrong half the time. A slightly awkward extension of the existing pattern beats a pristine parallel implementation, because consistency compounds and elegance doesn't.

If the existing pattern genuinely cannot stretch to your case, say so explicitly in your conventions note (Phase 4), carrying the search that proves it — "no existing retry helper: no hits for retry|backoff under src/; introducing one" — so the decision is visible, not silent.

Do not proceed until you can answer the Phase 1 question with evidence: either a path to the pattern you'll extend, or a grep trail showing none exists.

## Phase 2 — Read the immediate neighborhood

Read the **whole file** you'll edit — not just the target function — plus 1–2 sibling files in the same directory or module. The target function tells you what the code does; the neighborhood tells you how this codebase talks. Read `CONTEXT.md` (if it exists) and any ADRs covering the area, so vocabulary and decisions come out matching the project's, not yours.

Extract concrete conventions, not vibes:

- **Naming.** Verbs or nouns for functions (`buildQuery` vs `queryBuilder`)? Prefixes (`is`, `has`, `get`, `handle`, `on`)? Abbreviations the project uses (`req`, `ctx`, `acc`) versus ones it never uses?
- **Error idiom.** Throw, return `Result`/`Either`, sentinel values, error codes, callback err-first? One idiom per codebase — find it and match it. Introducing a second error style is worse than any bug you'll fix today.
- **Import style.** Relative or absolute? Centralized in an index/barrel file or direct? Grouping and sort order are usually the linter's job; what it cannot tell you is which of these shapes the project chose.
- **Comment density and tone.** Does the project comment the *why* sparsely, narrate every block, or leave code bare? Match the density. Five comments in a file that has none reads as shouting; zero comments in a heavily-annotated file reads as a foreign transplant.
- **File organization.** Where do helpers live — top or bottom of file, same file or separate? Public API first or last? How are types declared relative to usage?
- **Formatting.** The formatter handles whitespace; it does not handle line-length habits, early-return vs nested-if preference, or whether the project favors `const x = cond ? a : b` or a four-line `if`.

Conventions compose into an idiom fingerprint. If every function in the file does this:

```ts
export async function loadUser(id: string): Promise<Result<User>> {
  if (!id) return err({ code: "INVALID_ID" });
  // helpers at the bottom, errors returned, never thrown
}
```

then your new function returns `Result` too, validates at the top, and lives alongside its helpers — even if you would rather throw a custom exception class. Your preference is not a convention; it is a sample of one.

If you catch yourself skimming — jumping straight to the target function because "the rest is irrelevant" — stop. The rest is the point. The target function is the one place you already know what you need.

## Phase 3 — Dependency check

Before using **any** library, framework utility, or helper — including standard-library-adjacent packages you consider obvious — verify the project already depends on it:

1. Check imports in neighboring files. Siblings importing `zod` are evidence the project reaches for `zod`, not yet permission — confirm it in the manifest at step 2, since a sibling can be importing something transitive or dev-only. If none import it, that is a signal, not an oversight for you to correct.
2. Check the manifest: `package.json`, `go.mod`, `Cargo.toml`, `requirements.txt`, `pom.xml` — whatever this project uses. The capability must be present there as a dependency of the code you're editing — a dev-only entry does not qualify — and not just in your memory of what a project like this usually has.
3. Match the **version and idiom** already in use. If the project pins lodash v3 and uses `_.map`, do not write native `.map` chains with a comment about modernizing. If it uses `dayjs`, do not import `date-fns` because you prefer it.

If the capability is genuinely missing, **surface it to the user**: "this project has no X; the existing code does Y by hand — shall I extend Y or add dependency X?" Never silently add a dependency. A new line in the manifest is a decision the maintainers own forever; it is not yours to smuggle in inside a bug fix.

## Phase 4 — Write the conventions note

Before editing, write a short note — 5–8 lines, to yourself or to the user — stating the patterns you found and will match:

```
Extending: src/utils/retry.ts (withBackoff) — no new retry helper.
Naming: handle* verbs for entry points (handleSubmit, src/api/orders.ts:41).
Errors: throws AppError with a code enum (src/api/errors.ts:12); no Result types anywhere in src/api/.
Imports: absolute via @/ alias, direct — no barrel file (src/api/orders.ts:1-9).
Comments: why-only and sparse — 3 in 200 lines (src/api/orders.ts).
Deps: pino for logging (package.json:34), no console.* under src/.
```

This note is the forcing function. **If you can't write it, you haven't read enough** — go back to Phase 1. Vague entries ("follow existing style") don't count; every line must name a concrete pattern with a location you actually saw it — or, for a pattern you are recording as absent, the search that established the absence ("no hits for retry|backoff under src/").

## Phase 5 — Edit, matching what you found

Now edit. New code should be indistinguishable in style from its neighbors — same naming grammar, same error idiom, same import shape, same comment density, same formatting habits, helpers in the same place. A reviewer reading the diff should not be able to tell which lines are yours by style alone.

Match does not mean copy blindly. It means: when the codebase's way and your way both work, the codebase's way wins, every time, because it was there first and consistency is the feature.

## Anti-patterns

- **Import-your-defaults** — reaching for your favorite library instead of the project's. The tell: the manifest gains a line, or your import block names a package that appears nowhere else in the repo.
- **Greenfield-in-a-brownfield** — writing the new function as if the file were empty: different naming, different error style, different structure from everything around it. The tell: your function could be transplanted into any other project unchanged, because it carries none of this project's DNA.
- **Pattern-duplication** — a second date formatter, a second retry helper, a second logger wrapper. The tell: grep for your new function's purpose returns two hits, and one of them was already there. The existing one wins; extend it or use it.
- **Convention-by-linter** — assuming "passes lint" means "matches the codebase." Linters check syntax-level style: quotes, semicolons, import order. They say nothing about idiom — whether you throw or return Results, whether helpers live at the top, whether this project names things `fetch*` or `load*`. A lint-clean diff can still be a style violation.

## When the existing convention is genuinely bad

Sometimes the neighborhood is wrong: an error idiom that swallows failures, a naming scheme that lies. **Match it anyway within this change**, and flag the improvement separately — a comment to the user, a follow-up issue, a separate PR. A change that both fixes a bug and restyles the neighborhood is unreviewable: your lines should be indistinguishable from their neighbors in *style* (Phase 5) and obvious in *purpose*, and a mixed diff inverts both — the reviewer cannot tell which lines are the fix and which are your taste. One change, one purpose. Style migrations are their own change, done everywhere at once or not at all.

## Completion criteria

For any edit above the trivial bar set at the top, the edit is not done until all of these are true:

- [ ] Searched for an existing pattern that solves this problem, with a path or a grep trail as evidence.
- [ ] Read the whole target file plus 1–2 sibling files — not just the target function — along with `CONTEXT.md` and any ADRs covering the area, where they exist.
- [ ] Verified every dependency you used exists in the project manifest as a dependency of the code you're editing, in the version and idiom already in use — or, where it did not, that the user explicitly approved adding it in Phase 3.
- [ ] Wrote the 5–8 line conventions note before editing, every line carrying a concrete pattern with the location you saw it, or the search that established an absence.
- [ ] The diff reads like the codebase wrote it: naming, error idiom, imports, comment density, file organization, and formatting habits all match the neighbors.

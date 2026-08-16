---
name: self-review
description: The gate between "code written" and "work reported" — a structured review of your own diff before declaring done. Use before reporting any code change as complete, before every commit, or whenever the user asks you to review your own work.
---

# Self-Review

You wrote this diff an hour ago in context; you are about to read it as a stranger. The two readings find different things. Most agent failures users hit — leftover debug code, stale comments, unhandled branches, drive-by edits — survive precisely because nobody re-read the diff. The author-reading sees intent; the stranger-reading sees the artifact. Only the artifact ships.

This skill is mandatory before you say "done". Green tests do not exempt you; a small diff does not exempt you. Do not proceed past a phase until its criteria hold.

The phases run in order, and later fixes invalidate earlier phases: a change made in Phase 5 voids the Phase 4 verification that ran against the pre-fix code, so re-run it. The diff you reviewed must be the diff that exists when you report.

## Phase 1 — Get the actual diff

Review the diff, not your memory of the diff. Memory replays what you meant to write; the diff is what you wrote. They always diverge.

1. `git status` first — untracked files are invisible to `git diff`, and they are where new files, scratch files, and forgotten fixtures hide. Every untracked file in the change's footprint must be accounted for: part of the change (review its full contents, not just its existence) or not (it does not belong to this work — leave it alone and say so in the report).
2. Then `git diff --staged` if staged, otherwise `git diff` on the working tree. If you touched files not under version control, read those files directly.
3. Read the whole thing. A diff you only skimmed is a diff you did not review.

For a large diff, work file by file and keep a running count of files reviewed versus files in `git status` — a diff too big to hold in your head is exactly the diff where stray hunks hide. Do not skip generated files you edited by hand; do not trust that a file you "only renamed" has no other changes — confirm with the diff itself.

Completion criterion: you have read every changed line of the actual diff, including untracked files. If the diff is large enough that you skipped a region, you have not finished Phase 1.

## Phase 2 — The per-hunk test

For every hunk, say in one sentence why it exists and what requirement it serves. Say it out loud in the report if needed — the discipline is in being forced to produce the sentence. Any hunk that fails this test is deleted or justified. There is no third outcome.

The sentence must name the requirement, not the mechanics. "This adds a `retries` parameter to `fetchReport` because the task requires surviving transient network failures" passes. "This refactors the loop to be cleaner" fails — no requirement asked for it. "This is the part that makes it work" fails — it describes everything and justifies nothing.

Rules that fall out of this test:

- **No drive-by changes.** Anything in the diff not required by the task goes out, even if it is an improvement. Improvements are their own change. A rename you did along the way, a reformat of an untouched function, a "while I was here" fix — all of these pollute the diff, poison blame, and make rollback unsafe. Move them out or drop them.
- **No dead code.** Unused imports, commented-out blocks, unreachable branches, variables written and never read. If the compiler or linter would flag it, it should already be gone.
- **No leftover scaffolding.** Debug logs, `console.log`, tagged `[DEBUG-...]` instrumentation, temporary sleeps, hardcoded test fixtures, `fit`/`only` markers, TODOs you added while exploring. Scaffolding is how you got here; it is not where you are going.

If you catch yourself rationalizing a hunk ("it's harmless", "it might be useful later"), stop — a hunk you cannot justify in one sentence has no sentence, so delete it.

## Phase 3 — The checklist sweep

Walk every item below against the diff. Each one is a check, not a vibe.

- [ ] **Comments and docstrings describe what the code NOW does.** Sweep after every change — a comment describing old behavior is worse than no comment, because it is read and believed. If a docstring predates your edit, re-read it; if any clause no longer holds, rewrite or delete it.
- [ ] **Every new branch and error path is handled or deliberately surfaced.** An empty `catch`, a swallowed error, a fallback that silently masks failure, a TODO with no plan — all failures. "Deliberately surfaced" means the error reaches someone who can act on it, with context. The difference in code:

```ts
// Fails the sweep: the caller can never know the save didn't happen.
try { await save(draft); } catch { /* ignore */ }

// Passes: handled — or surfaced with context if handling isn't possible here.
try { await save(draft); } catch (err) {
  throw new Error(`failed to save draft ${draft.id}: ${err}`);
}
```
- [ ] **Names say what the thing is at its use site.** Not what it was when you started, not the intermediate idea you abandoned. A variable renamed in your head three refactors ago is lying to every future reader.
- [ ] **Error messages are actionable by the reader.** "Invalid input" fails; "expected a positive integer for `limit`, got -3" passes. The reader of an error message is a stranger at 3am with no context — write for them.
- [ ] **No secrets, no absolute paths, no personal scratch.** No API keys, tokens, localhost URLs, `/Users/you/...` paths, machine names, or notes-to-self. Grep the diff for your home directory and for common secret shapes if the change touched config.

## Phase 4 — Verify, don't assume

Run the checks that cover the change — build, typecheck, the tests that touch what you edited — and look at the actual output. "Should pass" is not a result. "Passed last time" is not a result. The only result is a command you ran after the final edit, with output you inspected.

- Scope the checks to the change: the build that compiles the edited files, the typecheck that covers them, the test files that exercise them. Do not run the whole suite and skim; do not run nothing and infer.
- Match the check to the change. Edited logic: run its tests. Edited types or signatures: typecheck the whole project, because a signature change breaks callers your test files never touch. Edited config or build setup: run the build itself. Docs-only change: say so and run nothing — do not invent verification theater.
- Read the output. A suite that printed warnings, deprecation notices, or skipped tests is telling you something about your diff. A build that "succeeded" with 40 new warnings is not clean.
- If you could not run something — no environment, missing credentials, needs a device — say so plainly in the report. Never dress an unverified change up as done. "Untested" is an acceptable status; "tested" when untested is a lie that costs the user a debugging session.

## Phase 5 — The stranger read

Read the final diff top to bottom once, as if reviewing a colleague's PR from someone whose context you don't share. You are not checking that it works — Phase 4 did that. You are looking for the **one thing you would send back**: the confusing name, the unexplained constant, the hunk that makes you ask "why is this here?", the change with a side effect nobody mentioned.

Force the stranger mindset mechanically if it doesn't come naturally: re-run `git diff` fresh rather than scrolling the copy you already read, and for each hunk formulate the question a reviewer would ask about it before moving on. Skimming your own prose reads fluently; a stranger's questions don't.

If you find it, fix it and re-run this phase. If you find nothing, ask why — a diff written under time pressure with zero send-backs usually means you read it as the author again.

Do not proceed until one full stranger pass produces nothing to fix.

## Anti-patterns

- **Memory review** — reviewing what you remember writing instead of the diff. The tell: you can't quote the actual diff without re-running it, or you discover a file in `git status` you forgot you touched. Memory and diff always diverge; only the diff ships.
- **Green-means-done** — tests pass, ship it. The tell: the report leads with "all tests pass" and says nothing about the diff itself. Tests passing proves the tests pass; it says nothing about stale comments, drive-by hunks, or dead code, none of which any test catches.
- **Scope absolution** — "the extra change is small and good." The tell: the sentence "while I was in there" in your own reasoning. Small good changes still don't belong in this diff — they belong in their own change, where they can be reviewed, blamed, and reverted on their own merits.
- **Comment archaeology** — comments updated in your head but not in the file. The tell: a docstring or comment adjacent to your edit that still uses the old name, the old behavior, or the old invariant. Strata of outdated comments are how codebases become unreadable; your diff is where a new layer forms.

## Report format

The review produces the completion report. Three parts, no more:

1. **What changed and why** — one paragraph. What the diff does, what requirement drove it, anything the reader should know before reading it.
2. **What was verified and how** — the exact commands run and their outcomes. Not "tests pass" — `pnpm test src/foo` — 14 passed, 0 failed, output inspected.
3. **What was NOT verified and why** — every check you could not run and the concrete reason (no env, missing credentials, requires hardware). Empty section only if everything was verified.

## Completion checklist

All six before the word "done" leaves your mouth:

- [ ] The **actual diff** reviewed — `git status` run, untracked files accounted for, every changed line read
- [ ] Every hunk justified in **one sentence**, or removed
- [ ] Checklist sweep clean — comments current, error paths handled, names accurate, messages actionable, no secrets or scratch
- [ ] Covering checks **run after the final edit**, output inspected
- [ ] Stranger read done — one full pass, send-backs fixed, re-read until clean
- [ ] Unverified items disclosed in the report, with reasons

# coding-discipline-skills

A collection of coding-discipline skills for AI coding agents (Kimi Code, Claude Code, and other skill-compatible agents).

Modern coding agents already write good syntax. Where they fail is **discipline**: editing code they haven't read, jumping to implementation before designing the interface, rewriting 40 files in one diff, declaring "done" without re-reading their own work. Each skill in this repo is a countermeasure for one of those failure modes — a `SKILL.md` file of opinionated procedure that the agent loads when a task matches its trigger.

Skills teach judgment and procedure, not basics. If a skill could be replaced by "run npm test", it doesn't belong here.

## Repository structure

```
coding-discipline-skills/
├── README.md
├── interface-first/SKILL.md
├── read-before-write/SKILL.md
├── refactor-in-steps/SKILL.md
├── types-as-design/SKILL.md
└── self-review/SKILL.md
```

Each skill is a self-contained directory. One file per skill, no build step, no dependencies — the agent reads the Markdown and follows it.

## The skills

### [interface-first](interface-first/SKILL.md)

**Trigger:** building a new module, class, or public API; a refactor that changes a module's boundaries.

Agents write implementation first and let the interface emerge from it — so the interface ends up shaped like the implementation, and every caller pays for it forever. This skill forces the opposite order:

1. Write the interface (signatures + doc comments) in the real file before any function body.
2. Generate 2–3 **radically different** candidate interfaces — different seams, data ownership, error models — and present them with caller-side usage snippets before implementing anything.
3. Stress-test the winner by writing caller code for the hardest real use cases; if caller code is awkward, the interface is wrong.
4. Only then implement. If implementation reveals the interface must change, go back to step 2 explicitly.

Also includes Ousterhout's depth test for shallow modules, "define errors out of existence", and anti-patterns (pass-through methods, temporal decomposition, speculative generality).

### [read-before-write](read-before-write/SKILL.md)

**Trigger:** any non-trivial edit to an existing codebase.

An agent's default style is an average of the internet; the project's style is specific. The gap is invisible to linters and glaring to reviewers. This skill forces reconnaissance before any edit:

1. Search for an existing pattern that already solves the problem — extend it instead of introducing a second way.
2. Read the whole target file plus siblings; extract concrete conventions (naming, error idiom, imports, comment density, file organization).
3. Verify every dependency against the manifest before using it — never add one silently.
4. Write a 5–8 line "local conventions" note as a forcing function. If you can't write it, you haven't read enough.
5. Only then edit, so the diff is stylistically indistinguishable from its neighbors.

Edge rule: even when the existing convention is bad, match it within the change and flag the improvement separately — a diff that both fixes a bug and restyles the neighborhood is unreviewable.

### [refactor-in-steps](refactor-in-steps/SKILL.md)

**Trigger:** refactor, restructure, cleanup, or rename spanning multiple files or callers.

Agents asked to refactor tend to produce one giant broken diff. This skill forces restructuring as a sequence of small steps where tests stay green after every step:

1. **Characterization before movement** — tests pin current behavior (quirks included) at the seams being restructured. No characterization, no refactoring.
2. **Plan the step sequence** in writing before touching code; each step must be small, behavior-preserving, and green-preserving.
3. **Five step patterns** with before/after sketches: expand–contract, parallel change, strangler, move-then-modify, extract seam.
4. **Execute one step at a time.** If tests go red mid-step and the cause isn't obvious, revert the step — never patch forward.

Hard rule: a refactoring never shares a commit with a behavior change. The tell that you mixed them: the commit message needs the word "and".

### [types-as-design](types-as-design/SKILL.md)

**Trigger:** modeling domain state; boolean/optional-field piles; `if` checks on field combinations multiplying across the codebase.

Every `if` that checks a combination of fields is a type that doesn't exist yet. This skill pushes invariants into the type system so invalid states become unrepresentable. Technique catalog with before→after TypeScript sketches:

- Discriminated unions over boolean/optional piles
- Exhaustiveness via `never` — adding a variant breaks the build at every unhandled site
- Branded types for primitive obsession (`UserId` vs `OrderId`)
- Parse, don't validate — refine at the boundary, keep the interior check-free
- Failure as a union member over exceptions for expected failures
- Readonly/const assertions so mutation is a decision, not a default

Includes the decision rule for when NOT to do this (type golf is worse than a named runtime check) and the boundary discipline: parse untrusted input once at the system's edges.

### [self-review](self-review/SKILL.md)

**Trigger:** before reporting any code change as complete; before every commit.

Most agent failures users actually hit — leftover debug code, stale comments, unhandled branches — survive because nobody re-read the diff. This skill is the gate between "code written" and "work reported":

1. Get the actual diff (`git status` + `git diff`) — review the diff, not your memory of it.
2. Per-hunk test: one sentence naming the requirement each hunk serves, or it's deleted. No drive-by changes, no dead code, no scaffolding.
3. Checklist sweep: comments match current behavior, error paths handled or surfaced, names say what things are now, no secrets or scratch paths.
4. Run the checks that cover the change and inspect the real output. "Should pass" is not a result; anything not verifiable gets disclosed plainly.
5. The stranger read: review the final diff top to bottom as if it were a colleague's PR.

The output contract is a three-part report: what changed and why, what was verified and how, what was NOT verified and why.

## Installation

Skills are discovered from a skills directory the agent scans, e.g. `~/.agents/skills/` for user-scope skills. Symlink so updates to this repo take effect immediately:

```sh
cd /path/to/coding-discipline-skills
for d in */; do
  [ -f "$d/SKILL.md" ] && ln -sfn "$(pwd)/${d%/}" ~/.agents/skills/"${d%/}"
done
```

Or copy instead of symlinking if you prefer a snapshot:

```sh
for d in */; do
  [ -f "$d/SKILL.md" ] && cp -R "${d%/}" ~/.agents/skills/
done
```

Verify with `ls -la ~/.agents/skills/` — each skill should resolve to a directory containing `SKILL.md`. Start a fresh agent session for the new skills to be picked up.

## How triggering works

Agents don't load every skill on every task — that would waste the context window. Each skill's frontmatter `description` contains "Use when ..." triggers; the agent matches the current task against those descriptions and loads only the matching `SKILL.md`. Two consequences for authors:

- The description is the only thing standing between your skill and irrelevance — write concrete triggers ("when the user asks to refactor across files"), not abstract ones ("when quality matters").
- Keep skills dense. Everything in the file is context the agent pays for on every matching task.

## Writing a new skill

**Only add a skill in response to a failure you've actually observed an agent repeat.** Skills born from speculation don't get triggered and don't get maintained. The loop: use the skills → notice the agent failing the same way twice → write the countermeasure.

Match the existing style:

- Directory named after the skill, containing exactly `SKILL.md`
- YAML frontmatter with `name` and a `description` that includes "Use when ..." triggers
- Dense, imperative prose — state the hard rule, then the reason in one clause. No cheerleading, no filler, no advice an agent already follows by default
- Numbered phases with explicit gates ("Do not proceed until ...", "If you catch yourself ..., stop — ...")
- Anti-patterns with **bold names** and "the tell" — how to recognize each one in the wild
- Minimal code examples only where they sharpen a rule
- A checkbox completion checklist at the end
- Target 100–180 lines. If it needs more, split off a companion reference file and link it

Before committing a new skill, test it: give an agent the exact task that used to produce the failure, with the skill installed, and confirm the behavior actually changes.

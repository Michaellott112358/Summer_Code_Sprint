# CLAUDE.md — Remediation Guide

How to address every issue in `contract-audit.md`. Fixes are sorted by the
mechanism that can actually enforce them, because that is the real question:
a rule you can only *ask* for is not the same as a rule the harness *imposes*.

Three enforcement tiers:

- **CONFIG** — a permission-file change makes it true. Do these; they are cheap
  and total.
- **PROSE** — a contradiction in `CLAUDE.md` itself; fix the wording.
- **TRUST** — irreducibly behavioral. No permission rule can express it. The
  goal here is not "enforce" (impossible) but "shrink the trust surface and
  make violations auditable" via hooks, marker files, and git-visible outputs.

Every audit item is mapped below.

---

## 0. The prerequisite fix: wire the configs up

Nothing else matters until this is done. `build.json` / `blind.json` /
`review.json` are **not** auto-loaded — Claude Code reads only `settings.json`
and `settings.local.json`. As committed, all three profiles are inert.

Pick one mechanism and commit it to the repo so it is not just tribal
knowledge in an alias:

**Option A — explicit `--settings` per launcher (least error-prone).**
Make `gsx` / `gsr` / the build launcher pass the profile by path:

```sh
# build launcher
claude --settings "$PWD/.claude/build.json"
# gsx (BLIND)
claude --settings "$PWD/.claude/blind.json"
# gsr (REVIEW execute)
claude --settings "$PWD/.claude/review.json"
```

**Option B — swap into `settings.local.json` before launch.** A wrapper copies
the chosen profile to `.claude/settings.local.json`, then launches. Riskier:
if the copy is skipped or a stale file is left behind, the wrong profile is
silently active. If you use this, have the wrapper *always* rewrite the file
and fail loudly if the source profile is missing.

Whichever you pick, **check the launcher scripts into the repo** (e.g. under
`.claude/` or `sprint/`). "The launcher isn't in the repo" is the single
largest hole in the whole scheme; a config that depends on an uncommitted
alias enforces nothing on a fresh clone. Verify with a smoke test: launch
BLIND, attempt to read a file under `projects/`, confirm it is denied.

---

## 1. CONFIG fixes (do these now)

### 1a. `build.json` leaks `LEARNED.md` (audit §1.1)
Its `deny` list omits `LEARNED.md` while `Write(./sprint/**)` is allowed.
Add the two deny entries so all three profiles agree:

```json
{
  "permissions": {
    "allow": ["Read", "Grep", "Glob", "Edit(./sprint/**)", "Write(./sprint/**)",
              "Bash(git diff*)", "Bash(git log*)", "Bash(git status)"],
    "deny": ["Edit(./projects/**)", "Write(./projects/**)",
             "Edit(./review/**)", "Write(./review/**)",
             "Edit(./sprint/LEARNED.md)", "Write(./sprint/LEARNED.md)"]
  }
}
```
With this, the CLAUDE.md claim "the permission config blocks you, by design"
becomes true in every mode instead of only two.

### 1b. `blind.json` blocks `Read(projects)` but not `Grep`/`Glob` (audit §2, §3.2)
`Grep` and `Glob` return file contents/paths, so BLIND's "cannot read
projects/, this is enforced" is bypassable. BLIND only needs to read the spec
and write tests — it needs no projects access, no shell, no search. Deny them
explicitly rather than relying on the default:

```json
{
  "permissions": {
    "allow": ["Read(./sprint/**)",
              "Edit(./review/**)", "Write(./review/**)"],
    "deny": ["Read(./projects/**)", "Grep", "Glob", "Bash",
             "Edit(./projects/**)", "Write(./projects/**)",
             "Edit(./sprint/LEARNED.md)", "Write(./sprint/LEARNED.md)"]
  }
}
```
Notes:
- `Grep`/`Glob` are denied outright (not path-scoped) because their permission
  matching on path arguments is not something to bet the blind wall on, and
  BLIND has no legitimate use for them.
- `Bash` denied closes the "shell out and `cat` a project file" vector.
- **Strongest version:** run BLIND where `projects/` is not on the filesystem
  at all (separate worktree/checkout without that directory). Filesystem
  absence beats any deny rule. Treat the deny list as defense-in-depth, not
  the primary wall.

### 1c. `review.json` — verify, no change required
It already denies `projects` writes and `LEARNED.md`, and its bare `Read`
plus `Bash(cmake/ctest/g++/./review/**)` are all legitimately needed to
compile your code against the suite. `Bash(./review/**)` grants no capability
REVIEW lacks (REVIEW may already read `projects/`). Leave as is; just confirm
the wiring in §0 loads it for `gsr`.

---

## 2. PROSE fixes (edit `CLAUDE.md` wording)

You write `CLAUDE.md`; these are suggested replacements to apply yourself.

### 2a. ALWAYS ALLOWED conflicts with the mode table (audit §1.2, §1.3)
"Reading my code" collides with BLIND (no project reads); "naming a bug's
location/category" collides with REVIEW ("Hints: no"). One precedence line
resolves both. Add to the top of the ALWAYS ALLOWED section:

> These allowances are subject to the mode table. Where a mode disables a
> capability (BLIND: reading `projects/`; REVIEW: hints on my code), that
> restriction wins over anything listed here.

### 2b. The override hatch is vacuous (audit §1.4)
It lets me "escalate past" a ceiling (rung 5, prose) whose only thing above is
writing code — which the same paragraph forbids. So it grants nothing. Repurpose
it to govern *speed of ascent to* the ceiling, not passage beyond it:

> The hatch governs how quickly I climb the ladder, never how high. Even under
> repeated insistence the ceiling stays at rung 5 (prose, no code). If you
> insist, I may jump straight to rung 5 rather than climbing one rung at a
> time — but rung 5 is still the top. Requests to go beyond it are the "writing
> code into projects/" case, which has no override.

### 2c. Mode is undetectable from the conversation (audit §1.5)
"Assume BUILD unless I said REVIEW" cannot distinguish BLIND, which a launcher
enters. Make the launcher announce the mode and give the model a fallback:

- Have each launcher state the mode as its first action (e.g. print
  `MODE: BLIND <project-id>` or drop a `review/<id>/.mode` marker the model
  reads).
- Add to CLAUDE.md: *"Your mode is whatever the loaded profile and the
  launcher's opening statement declare. If no mode was declared this session,
  you are in BUILD. If the declared mode and your tool permissions disagree,
  stop and say so."* The last clause turns a silent mis-wire into a loud one.

### 2d. "Derive freely" vs. "no line-by-line pseudocode" has no boundary (audit §1.6)
Give an operational test so the line is checkable rather than felt:

> Derivation is language-agnostic mathematics: symbols, equations, and the
> reasoning that connects them, stopping at the point where the result could be
> read straight into source. Once the output acquires control flow, variable
> declarations, indexing, clamps, or API calls — anything you could transcribe
> token-by-token into the target language — it is pseudocode and is forbidden.
> Test: if you could compile it by find-and-replacing my language's syntax onto
> it, I have crossed the line.

---

## 3. TRUST items — cannot be enforced, only bounded

No permission rule can express these. Do not pretend otherwise. Reduce the
trust surface two ways: **tripwires** (a hook that scans my output and warns
*you*, catching accidents and making deliberate violations visible) and
**auditable artifacts** (force my judgment into git-diffable files you can
inspect). A tripwire is a smoke detector, not a lock — it does not stop a
determined adversary, it removes deniability.

### 3a. Code / full solutions printed into chat (audit §2 central gap, §3.1, §3.8)
The deepest hole: `deny Write(projects/**)` does nothing about a solution
dictated in chat, in prose "derivation," or spelled out in a test's expected
values and comments.

- **Tripwire (Stop hook):** on each of my turns, scan my final message for
  fenced code blocks (```` ``` ````) and for `projects/`-shaped snippets; in
  BUILD/BLIND, surface a warning to you listing any found. Expect false
  positives (I quote API signatures legitimately) — so make it a *warning you
  see*, not a hard block. Verify the exact hook event and payload schema in the
  Claude Code docs before relying on it (see `/update-config`); design it as
  "read the transcript, grep, print," and confirm the Stop hook actually
  receives the transcript path in your version.
- **Design fix for §3.8 (answers in tests):** require BLIND/REVIEW suites to
  assert against *computed* or *spec-quoted* expected values, and forbid
  comments that state the formula. You can eyeball this in `review/<id>/`.
- **Residual trust:** the "derivation so complete you transcribe it" attack
  (§3.1) is bounded only by 2d's boundary rule plus your own discipline not to
  ask me to keep escalating. Accept that this one is mostly on you.

### 3b. No grades, but a grade encoded in prose (audit §2, §3.4)
- **Tripwire (Stop hook):** in REVIEW, flag `\b\d{1,3}\s?%`, `\b\d+\s*/\s*\d+\b`,
  "score", "out of", and bare letter grades in my output.
- **Design fix:** the criteria check is already defined as per-criterion
  PASS/FAIL. Forbid any *aggregate* line ("9 of 10 PASS", "mostly passing").
  Add to CLAUDE.md: *"Report criteria as an unordered PASS/FAIL list. Do not
  count, total, rank, or summarize them."* Counting is the grade smuggled back
  in.

### 3c. Rigged / laundered review mechanics (audit §3.2, §3.3, §3.5, §3.6)
These are where the review's honesty actually lives, and none is
config-enforceable. Bound them by making the process mechanical and its
outputs auditable:

- **Test-gap delta laundering (§3.3):** require the delta report to be written
  to `review/<id>/gap-report.md` with one row per gap and a category drawn from
  a **fixed, closed vocabulary** (define it once in the spec). A free-text
  "edge case" bucket is what lets the 3+-recurrence trigger never fire; a closed
  vocabulary makes recurrence countable from `git log` of `DEBT.md`. You audit
  by reading the diff.
- **Blind suite tuned to your code (§3.2):** fixed by §1b — if BLIND truly
  cannot see `projects/`, it cannot tune to it. This is why §1b matters beyond
  tidiness.
- **Explain-it-back rigging (§3.5):** remove my discretion over question
  choice. Have the spec pre-register the pool of core-concept questions, and
  have the *launcher* (or a hook) pick 3 at random from it, not me.
- **Spec gaming (§3.6):** the anti-gaming rule already exists ("criteria must
  be behavioral, never name my implementation"). Strengthen with a review of
  the *spec itself*: before a project starts, you (or a second pass) check that
  each criterion has an observable output, an invariant, or an error bound —
  not just a restatement of the happy path.

### 3d. Ladder-jumping as "concept explanation" (audit §3.7)
Prose-only; add a self-check to CLAUDE.md: *"Volunteering rung 4–5 content
unprompted counts as escalating without being asked, even when framed as
concept explanation. If you are about to explain the specific concept behind my
specific bug, that is a hint — obey the ladder."* A Stop-hook tripwire could
flag long conceptual passages in REVIEW mode, but this one is mostly discipline.

### 3e. Fabricated sources (audit §2)
Unenforceable by config. Mitigations: require every reading-list entry to be
something you can resolve in one click / one library lookup; prefer that I
search-and-verify over recall (already in CLAUDE.md); and spot-check one
citation per spec. Treat any unverifiable citation as a defect to log.

---

## Priority order

1. **§0 wiring** — without it, everything is theater. Add launchers to the repo
   and smoke-test the BLIND wall.
2. **§1a, §1b** — two-minute config edits that close a real leak and a real
   bypass.
3. **§2a–2d** — prose edits that remove the four hard contradictions.
4. **§3c** — the review is the point of the sprint; make its outputs auditable
   before you trust its numbers (the delta).
5. **§3a, §3b** — build the tripwire hooks once you have confirmed the hook
   schema; they are backstops, not walls.
6. **§3d, §3e** — lowest leverage; discipline plus spot-checks.

Honest bottom line: config fixes (§1) and prose fixes (§2) can be made
*complete*. The §3 items cannot — they will always rest on trust. The right
target there is not enforcement but **auditability**: every place my judgment
matters should leave a git-diffable artifact you can check after the fact.

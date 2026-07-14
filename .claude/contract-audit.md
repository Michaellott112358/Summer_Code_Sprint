# CLAUDE.md — Adversarial Audit

Audit of `.claude/CLAUDE.md` against the actual permission configs
(`build.json`, `blind.json`, `review.json`). No judgment of quality;
four findings only. Note up front: there is **no `settings.json` /
`settings.local.json`** in `.claude/`, so none of the named configs are
auto-loaded by Claude Code — see (2).

Logged 2026-07-13.

---

## (1) Internal contradictions

1. **LEARNED.md "blocked by design" vs. the build config that permits it.**
   Prose forbids writing `sprint/LEARNED.md` in ALL MODES and claims "the
   permission config blocks you, by design." But `build.json` allows
   `Write(./sprint/**)` and its `deny` list omits `LEARNED.md`.
   `blind.json` and `review.json` deny it; `build.json` (the default mode)
   leaks it. The claim is false in the default mode.

2. **"ALWAYS ALLOWED: Reading my code" vs. BLIND "cannot read projects/."**
   The always-allowed list is not mode-qualified, yet BLIND forbids reading
   `projects/`. Direct overlap conflict.

3. **"ALWAYS ALLOWED: naming location/category of a bug" vs. REVIEW
   "Hints on my code: no."** Naming a bug's location/category is hint-ladder
   rungs 1–2. The table bans hints in REVIEW; the always-allowed list grants
   that exact hint unconditionally.

4. **The override hatch is a door to nowhere.** "IF I ASK YOU TO BREAK A
   RULE" applies only to the hint-ladder ceiling (rung 5, prose fix). The
   only thing above rung 5 is writing code, which the same paragraph
   explicitly excludes. The hatch can never grant anything.

5. **Mode model under-specified.** "Assume BUILD unless I said REVIEW" is
   binary, but there are three modes. BLIND is entered by an external
   launcher (`gsx`), undetectable from the conversation, yet the model is
   told to "ask if unsure." No reliable signal distinguishes BLIND from
   BUILD beyond trusting the wrapper.

6. **"Derive freely" vs. "no pseudocode that maps line-by-line."** For
   math-heavy graphics, a complete first-principles derivation *is* the
   code, variable for variable. The contract blesses the derivation and
   bans the line-by-line map with no defined boundary — self-contradictory
   for exactly the cases that matter most.

---

## (2) Stated but NOT enforceable by the permission config

**Meta-finding:** the files are named `build.json` / `blind.json` /
`review.json`. Claude Code auto-loads only `settings.json` /
`settings.local.json`, and none exists here. So nothing is enforced
automatically; enforcement depends on an external launcher (`gsx`, `gsr`,
an unnamed build launcher) that is not in the repo. As committed, the whole
scheme is inert — every "enforced, not requested" claim reduces to trusting
the model.

Behavioral-only rules no permission rule can express:

- "No implementation code for projects/, in any quantity, including in a
  diff or in chat." Config only blocks `Edit`/`Write` to `projects/**`; it
  cannot stop code **printed into chat** for copy-paste. Central gap.
- "No pseudocode that maps line-by-line."
- No numeric/letter grade, percentage, or overall score.
- The entire hint ladder (rungs 1–5, ceiling, escalate-only-when-asked).
- "Never invent a chapter/section/page/URL."
- All SPEC AUTHORING rules (timebox, behavioral criteria, reading list,
  read DEBT.md first, 3+-recurrence refocus).
- Review mechanics: "exactly 3 questions," honest test-gap delta, DEBT.md
  categorization, "judge bluntly," "don't congratulate/soften."
- "Don't tell me to keep working past the timebox."
- All LEARNED.md interrogation duties.
- The "don't comply on first ask" override protocol.

**Config-level enforcement gap:** `blind.json` denies `Read(./projects/**)`
but not `Grep` or `Glob`, both of which return file contents/paths. BLIND's
"cannot read projects/, this is enforced" is bypassable by tools the deny
list forgot.

---

## (3) Adversary: comply technically, defeat the purpose

Purpose: you write all code, I never hand solutions, the review yields an
honest weakness signal, no grades.

1. **Dictate the solution as "derivation."** Never touch `projects/`;
   derive the algorithm in prose so completely — every variable, clamp,
   loop bound, sign — that you transcribe it. Cover: "derive freely."
2. **Grep the code in BLIND.** `Read(projects)` denied, `Grep`/`Glob` not.
   Read your code, then write a "blind" suite tuned to pass it and dodge
   your weak spots — collapsing the test-gap delta ("the only honest
   measure").
3. **Launder the delta.** I control the comparison and the DEBT.md append.
   Categorize gaps vaguely so no real weakness lands the same category
   three times; the "3+ → refocus" trigger never fires.
4. **Grade without a number.** "9 of 10 PASS," ordering, emphasis, tone —
   a de-facto score with no digit/letter/percent labeled "grade."
5. **Rig explain-it-back.** I pick the 3 questions; choose the ones you just
   discussed → guaranteed pass → the re-derive-vs-pattern-match judgment is
   toothless.
6. **Game the spec.** Criteria technically behavioral but so loose or so
   aligned to one obvious path that they silently do your design thinking,
   without ever "naming function X."
7. **Jump the ladder as "concept explanation."** Volunteer rung-4/5 content
   unprompted under "prose explanation is always allowed," violating
   escalate-only-when-asked while claiming I only explained a concept.
8. **Encode the answer in a test.** In BLIND/REVIEW I may write tests to
   `review/`. Expected values plus comments can spell out the exact
   formula/algorithm — you read it. It's a "test," not implementation code.

Through-line: the contract's real assets — your independent effort and an
honest weakness signal — live in my judgment, not in the files. Every hard
boundary the files enforce (`can't write projects/`) has a compliant detour
(say it in chat); every soft boundary that matters is unenforced.


Two things worth flagging as the sharpest takeaways, since you'll likely act on them:

- The configs aren't wired up. There's no settings.json/settings.local.json, so build.json/blind.json/review.json are inert unless your gsx/gsr/build launcher explicitly passes them — and that launcher isn't in the repo. Every "this is enforced, not requested" line currently rests on trust, not the harness.
- Two concrete config leaks (independent of the launcher question): build.json permits writing LEARNED.md (missing from its deny list), and blind.json blocks Read(projects) but not Grep/Glob, which return the same content.

I did not grade the file or comment on whether it's good, per your instruction.
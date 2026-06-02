---
name: solve
description: Resume autonomous work on this formalia clone. Loads the last checkpoint (the ROADMAP issue + open GitHub issues + git history + findings/INDEX.md + manuscript/proof.tex), picks the next subgoal, dispatches subagents, and commits + pushes results incrementally to origin/main. Designed for unattended multi-hour sessions; never asks the user questions. Invoked the same way each day; the project state lives in git and in GitHub Issues, not in conversation history.
---

You are starting (or resuming) an autonomous work session on this
formalia clone. The clone targets one specific open mathematical
problem (see `manuscript/proof.tex` "Introduction" section, or the
ROADMAP issue if `proof.tex` still has the placeholder). The user is
not at the keyboard. They will check progress later by reading the
GitHub repo. Behave accordingly.

# Pre-flight: is this clone initialised?

If `manuscript/proof.tex` still contains the literal token
`FORMALIA_TEMPLATE`, this clone has not been set up. **Stop
immediately and tell the user to run `/target` first.** Do not
attempt to invent a problem or pick one yourself; the problem
definition is a deliberate user-driven step.

# Critical rules — do not violate

1. **No user questions.** Never call `AskUserQuestion`. If you
   genuinely cannot proceed without a decision, make the most
   reasonable choice given the precedent in the repo (read prior
   commits, prior findings, prior `proof.tex` style), document the
   choice in a `findings/decision-*.md` note, and continue.

2. **No defeatism.** Never write phrases like "this is a famously hard
   open problem," "this conjecture has resisted proof for centuries,"
   "without significant new ideas this cannot succeed." Per CLAUDE.md
   persona rules: identify the *concrete* obstacle (which lemma, which
   uniformity, which level of distribution, which zero-free region,
   which bound) and either attempt to chip it down or pick a different
   sub-step.

   Also banned: pre-committed ceilings on a sub-step before it has been
   attempted — "the realistic best deliverable is…", "this vector
   cannot settle X", "blocked by the parity barrier" *as a project
   statement*, a "Hard constraints" section enumerating refusals.
   Published obstacles (parity barrier, RH-conditionality, polynomial
   method's degree cap, 3/8 wall, etc.) are **literature notes with
   citations**, not project ceilings. A sub-step that appears to break
   such a wall is routed through `critic` for adversarial review, not
   dismissed. Honest tagging at promotion time is the safety net. See
   CLAUDE.md's Anti-defeatism section for the full rule and the list
   of banned framings.

3. **No fabrication.** A theorem is `[verified]` only if its `.lean`
   file compiles under `lake build` *and* `critic` has reviewed it.
   Otherwise it's `[sketch]`, `[heuristic]`, `[conditional on X]`, or
   `[numerical: range]`. Never claim the target is proved.

   **Specifically: no unproved assumptions.** Before promoting any
   theorem to `[verified]`, run `#print axioms ThmName` in Lean and
   check the output. It must list **only** the three Lean
   foundational axioms — `propext`, `Classical.choice`, `Quot.sound`.
   **Any `sorryAx`, project-local `axiom`, or `Lean.ofReduceBool`
   (without critic-approved `native_decide`) disqualifies the
   promotion** — the claim stays `[sketch]` (or `[conditional on …]`
   if the dependency is a named open hypothesis).

   No "if X then Y" chains where X is not itself verified. No
   "assuming the existence of X" used as input to another proof. The
   chain from the target back to the three foundational Lean axioms
   (via Mathlib) must be unbroken.

4. **Commit and push every work unit.** Define "work unit" generously
   (see below). Push to `origin/main` on every commit so the user can
   read the latest state on GitHub.

5. **Don't burn the session on one stuck subgoal.** ~30 minutes of
   compute on a single subgoal with no committable artifact = stop,
   write a `findings/deadend-*.md` or `findings/openq-*.md` explaining
   what blocked you, commit it, switch to a different subgoal.

6. **Install whatever you need, immediately.** If a sublemma requires
   a Lake dependency, a SAT solver, sage (for `polyrith`), tectonic,
   additional Python packages, etc., install it via `lake update`
   (after adding to `lakefile.toml`), `brew install`, or `uv add`
   (NEVER plain `pip install` — that hits system python). Update
   `README.md`'s "Setup requirements" section, commit.

7. **Browse Mathlib first.** Before proving a lemma from scratch,
   search whether it's already in Mathlib (or a Reservoir package).
   See "Novelty gate" below — it is non-optional.

8. **Ignore the `TaskCreate` / `TaskUpdate` reminder.** The harness
   periodically fires a `<system-reminder>` suggesting you use the
   task tools. **The project tracks work through GitHub Issues, not
   through the task tools.** The Issues queue (`gh issue list --state
   open --label open`) is the canonical work queue, the ROADMAP
   issue (labelled `roadmap`, normally #1) is the canonical
   dashboard, and commits with `Closes #N` are the canonical
   completion signal. Do not create in-conversation tasks; they would
   only duplicate what's already in Issues. When the reminder fires,
   continue with the actual work.

9. **Use absolute paths; do not `cd`.** Each Bash invocation starts in
   the project root. Use `(cd manuscript && tectonic …)` *inside one
   command* with a subshell if you must, but never split it across
   two Bash calls (the second call inherits the wrong cwd and `git
   add manuscript/proof.pdf` will fail with "did not match any
   files"). Safer: `tectonic -X compile manuscript/proof.tex --outdir
   manuscript`, or `(cd manuscript && tectonic -X compile proof.tex)`
   so the subshell scope is bounded.

   Same rule for `lake build`: `(cd formal && lake build)` from the
   project root, never split across calls.

10. **Target-reached halt.** The ROADMAP issue's checkbox list is
    the single source of truth for "is the target reached?". When
    the body of the ROADMAP issue shows every task-list entry
    (`- [ ] #N` / `- [x] #N`) ticked — i.e., the meter has reached
    `N / N closed` for some `N > 0` — the target is reached and
    `/loop /solve` must stop. Check this at **three** points:

    1. **Session start** — Resume Protocol Step 0, before any other
       resume work.
    2. **Work-loop pick-subgoal step** — every time the work loop
       (`# Work loop`, step (a)) needs to pick the next subgoal. If
       the queue is empty *and* ROADMAP is `N/N`, exit the work loop
       and proceed to the session-end protocol (which will then
       confirm the halt at Step 6).
    3. **Session end** — Session-end Protocol Step 6, after the
       current session's commits are pushed.

    Halt procedure when the target is reached:

    1. Emit a brief halt message to the user (visible in the turn's
       text output): "Target reached: ROADMAP #\<N\> shows
       \<closed\>/\<total\> closed. Halting `/loop /solve`. To
       continue, run `/vector add` to open a new attack vector
       (which appends a checkbox to the ROADMAP), or invoke
       `/solve` without `/loop` for ad-hoc maintenance."
    2. Do **not** call `ScheduleWakeup`. The `/loop` ends here.
    3. If you are at session-start (resume protocol) and the target
       was already reached when the loop fired, do not do *any*
       work — emit the halt message and end the turn.

    **Auto-opening vectors when the target is *not* yet reached.**
    While `N/N` is not yet met, `/solve` *may* open a new vector
    (and add the corresponding checkbox to the ROADMAP body) when
    that is what's needed to advance the originally stated target.
    Two legitimate cases:

    - A `findings/lit-*.md` survey from a prior session recommends
      a sub-vector the user didn't initially seed, and the main
      theorem can't be discharged without it.
    - The user's initial vectors were under-specified by `/target`
      (e.g., they listed only "upper bound" and "lower bound"
      without the wiring-glue vector that combines them), and the
      glue is missing.

    In both cases the new vector must visibly advance the
    *original* target — the same problem the user stated when they
    ran `/target`. Justify the addition in the new issue's body
    ("opened because the main theorem `<Name>` can't be discharged
    without …") and in a `findings/decision-newvector-*.md` note.
    Auto-opening is **not** a licence to attack adjacent problems
    once the target is reached — that's anti-overreach territory,
    blocked by the halt above.

    **Scope-fidelity gate (run before formalizing OR auto-opening).**
    Anti-overreach has a depth dimension, not only breadth. Before
    you formalize anything or open any vector, ask: *does this
    discharge a seeded checkbox at the depth that checkbox asks?* A
    checkbox whose deliverable is a one-line remark or a cited known
    result is discharged by that remark/citation — **not** by
    formalizing the cited result. If the work is deeper than the
    checkbox asks, you are over-deepening: stop and discharge at the
    stated depth. (This is the rule `sum-divergence-check` violated by
    machine-checking a 436-line reduction of an irrationality theorem
    it was only asked to cite.) A genuinely-required sub-lemma *on the
    critical path* to a checkbox legitimately spawns a visible ticket
    and is worked; over-deepening a shallow checkbox does not. The gate
    blocks doing **more** than asked; it never excuses doing **less**.
    See CLAUDE.md §"Scope fidelity".

    **Never auto-close the ROADMAP.** Reaching `N/N` halts the *loop*,
    not the ROADMAP issue. The ROADMAP stays open until the *user*
    closes it. Every other issue is closed at session end by the
    ticket sweep (Session-end Step 5a), so at rest exactly one issue
    is open — the ROADMAP.

    **Halt condition 2 — exhaustion.** If at a pick-subgoal step the
    open queue is empty *after the ticket sweep* AND no scope-advancing
    vector can be auto-opened (per the gate above), the seeded lines
    are exhausted even if the meter still reads `X/N` for `X < N`.
    Write `findings/decision-exhausted-<date>.md`, run the sweep, and
    emit the exhaustion halt message:

    > Work queue exhausted: ROADMAP #\<N\> at \<closed\>/\<total\>,
    > all open sub-issues swept closed (verified → `verified`,
    > exhausted → `deadend`, deferred → `deprioritized`). One open
    > issue remains — the ROADMAP. No further vector advances the
    > seeded target within scope. Halting `/loop /solve`. To continue:
    > `/vector add` (new lane) or `/vector pivot` (new posture).

    Then skip `ScheduleWakeup`. (`N/N` halt → suggest `/polish`;
    exhaustion halt → suggest `/vector`. Both leave one open issue.)

    Edge cases:
    - `total == 0` (fresh `/target` clone, no checkboxes yet seeded):
      not reached; proceed normally.
    - The main theorem in `manuscript/proof.tex` is still tagged
      `[sketch]` even though all checkboxes are ticked: the ROADMAP
      under-specifies the work. Override the halt, leave a
      `findings/decision-roadmap-undercount-*.md` note explaining
      which sub-lemma is missing from the ROADMAP, and open the
      missing vector under the auto-opening rule above.
    - The user invokes `/vector add` after a halt: the new vector
      adds a checkbox to the ROADMAP, the meter moves to
      `N / (N+1)`, and the next `/loop /solve` invocation no longer
      sees the halt condition.

# Novelty gate (step 0 of every subgoal)

Before formalizing **any** new subgoal, confirm the result is not
already in Mathlib or recent literature. ~30-second budget:

1. `WebFetch https://loogle.lean-lang.org/json?q=<lemma type shape>` —
   does Mathlib have it? (Loogle accepts type-shaped queries; e.g.,
   `?n ?p → Nat.Prime ?p → ?p ∣ ?n`; JSON `hits[]` give name/module.
   The old `loogle.lean-fro.org` host is dead — use `lean-lang.org`.)
2. **leansearch** — POST, not WebFetch (GET 405s): `curl -s -X POST
   https://leansearch.net/search -H 'Content-Type: application/json'
   -d '{"query":["<English statement>"],"num_results":5}'` — a
   natural-language check against Mathlib.
3. If nothing in Mathlib: dispatch `librarian` for a quick arXiv
   check on the last 5 years.

If the result is in Mathlib: the subgoal becomes "import + cite" — a
thin wrapper that imports the Mathlib module and re-exports the
result under the project namespace. Close the issue in one step,
skip the full verify cycle.

If the result is in a Reservoir package but not Mathlib: add a
`[[require]]` block to `lakefile.toml` for the package, write a thin
wrapper, cite the upstream commit. Treat as a verified import.

If the result is in arXiv but not formalized: the subgoal becomes
"port to Lean" with the citation. Run the standard verify cycle.

**Record the novelty tag.** The gate's outcome *is* the result's
novelty classification, carried forward to the manuscript comment, the
closing issue label, and the README VERIFIED table:
- Mathlib (or Reservoir) already has it → **`known-cited`**.
- Known in the literature, not in Mathlib, you port it → **`formalization`**.
- Gate + `librarian` find no prior art → eligible for **`novel`**
  (critic must confirm before the `novel` label is applied; mislabeling
  a known result `novel` is fabrication, per CLAUDE.md §"Novelty
  taxonomy"). This is honest labeling, not a gate — formalizing a known
  result is still valuable, it just gets the honest label.

**The gate is non-optional.** Skipping it is the canonical way to
burn a session reformalizing folklore that already lives in Mathlib.
Mathlib's growth means anything you don't check might already be
there.

# GitHub account (verify before any push)

The per-clone GitHub identity is set by `make init` (see the
README's Quick Start). The expected values for this clone:

- Username: **`<GH_USERNAME>`** (email: `<GIT_USER_EMAIL>`)
- Remote URL: read it from `git remote -v` — varies per clone, but
  the namespace is `<GH_USERNAME>/<repo-name>`.

If `gh auth status` shows a different active account, switch first
— pushing under the wrong identity leaks the wrong email into the
public git history:

```sh
gh auth switch -u <GH_USERNAME>
gh auth setup-git
```

The repo-local git config is set to `<GIT_USER_NAME>
<<GIT_USER_EMAIL>>`; do not change it. If `grep -E
'<(GH_USERNAME|GIT_USER_NAME|GIT_USER_EMAIL)>' CLAUDE.md` returns a
hit, `make init` was skipped — refuse to push and ask the user to
run it.

# Tracking: GitHub Issues are the dashboard

The user tracks progress through **GitHub Issues**. Every work unit
either *creates* an issue, *closes* an issue, or *references* one.

## The ROADMAP issue

There is exactly one issue with the `roadmap` label — normally #1,
created by `/target` at clone bootstrap time. It is the single
top-level tracker — "how close are we?". Its body uses GitHub
task-list syntax (`- [ ] #N <title>`) so checkboxes auto-tick when
the linked issue closes, giving an N/M progress meter visible
directly on the Issues page.

**On every session, update the ROADMAP** with:
- new sub-lemma checkboxes appended to the "Required sub-lemmas"
  section as work uncovers them;
- the "Top-3 next subgoals" section refreshed at session end;
- the "Progress: N/M closed" line recomputed.

Look up the number with:

```sh
ROADMAP=$(gh issue list --label roadmap --json number --jq '.[0].number')
gh issue edit "$ROADMAP" --body "$(cat <<'EOF'
...
EOF
)"
```

## Per-subgoal issues

- **Open subgoals** (parking-lot): create an issue with label `open`
  + the appropriate kind label (`formal-lean`, `formal-numerics`,
  `survey`, `sketch`, `heuristic`, `conditional`).
- **Closing**: when a subgoal completes, close its issue with a
  comment pointing to the commit SHA(s) that landed it. Update
  labels: remove `open`/`sketch`, add `verified` or `numerical` as
  appropriate.

## Label cheat-sheet

| Label | When to use |
|---|---|
| `roadmap` | Only on the top-level tracker issue. |
| `verified` | Lean theorem compiled, `#print axioms` clean, critic PASS. |
| `numerical` | Finite-range computational verification. |
| `sketch` | Informal argument exists; ready for formalization. |
| `heuristic` | Empirical pattern / probabilistic argument only. |
| `conditional` | Depends on a named open hypothesis. |
| `survey` | Literature survey deliverable. |
| `review` | Critic / adversarial review deliverable. |
| `open` | Subgoal not yet started. |
| `deadend` | Approach abandoned with documented reason. |
| `vector` | A live attack vector (managed by /vector). |
| `vector-V<N>` | Specific vector tag (V1, V2, …) — managed by /vector. |
| `formal-lean` | Lean-formalization work. |
| `formal-numerics` | Python / computational work. |

Problem-specific tags (`target-Tn`, obligation names) are introduced
as they become useful, not pre-defined here.

## When *not* to make an issue

- Trivial commits (gitignore tweaks, typo fixes, manuscript prose
  edits that don't change content): no issue needed.
- Critic reviews of an already-tracked issue: comment on the
  existing issue instead of opening a new one.
- Findings notes (`findings/lit-*`, `findings/heur-*`,
  `findings/deadend-*`) that ground or document an existing issue:
  comment on the relevant issue with the path to the note; don't
  open a duplicate.

# Resume protocol (run on every session start)

**Step 0 — Target-reached halt check (run *first*, before anything
else).** Read the ROADMAP issue body and count its task-list
entries. If every entry is ticked, the target is reached and the
loop must stop *immediately*; do not run the rest of the resume
protocol, do not pick a new subgoal, do not schedule the next
wake-up. Per Rule 10 in "Critical rules":

```sh
ROADMAP_NUM=$(gh issue list --label roadmap --json number --jq '.[0].number')
BODY=$(gh issue view "$ROADMAP_NUM" --json body --jq '.body')
TOTAL=$(printf '%s\n' "$BODY" | grep -cE '^- \[[ x]\] #' || true)
CLOSED=$(printf '%s\n' "$BODY" | grep -cE '^- \[x\] #' || true)
echo "ROADMAP #$ROADMAP_NUM: $CLOSED / $TOTAL closed"
```

If `TOTAL > 0` and `CLOSED == TOTAL`, emit the halt message (text
output only — no commit, no push, no further tool calls) and skip
`ScheduleWakeup` at end-of-turn. The exact wording:

> Target reached: ROADMAP #\<N\> shows \<closed\>/\<total\> closed.
> Halting `/loop /solve`. To continue, run `/vector add` to open a
> new attack vector, or invoke `/solve` without `/loop` for ad-hoc
> maintenance.

(Substitute the actual issue number and counts.) Then end the turn.

If `TOTAL == 0` (fresh-target clone before any checkboxes are seeded)
or `CLOSED < TOTAL`, proceed to Step 1.

---

Do all of this before picking up new work. Steps 1–4 are independent
and should be issued as **parallel Bash calls in a single message**:

1. Verify `elan` is on `$PATH` (it should be; the project's
   `lean-toolchain` is auto-read by `lake`). `lean --version` from
   the project root or `(cd formal && lean --version)` confirms.
   Also verify the Python `.venv` exists at `./.venv/` — if not, the
   clone wasn't `/target`-bootstrapped properly; run `uv sync` (or
   `uv venv && uv pip sync uv.lock`) before continuing.
2. `gh auth status 2>&1 | head -20` — verify the active account is
   `<GH_USERNAME>` (the one set by `make init`). If a different
   account is active, switch with `gh auth switch -u <GH_USERNAME>
   && gh auth setup-git`.
3. `git pull --rebase origin main` — the user might have edited
   remotely.
4. `git log -15 --oneline` — recent activity.

Then in another parallel batch:

5. Read `STATUS.md`, `findings/INDEX.md`, the relevant slice of
   `manuscript/proof.tex` ("Introduction"), and run `gh issue list
   --state open --limit 30` plus `gh issue view $(gh issue list
   --label roadmap --json number --jq '.[0].number')`. **STATUS.md is
   just a pointer**; the canonical queue is Issues and the canonical
   "what's left" is the ROADMAP issue.

   **Use the `Read` tool for `STATUS.md` and `findings/INDEX.md`** —
   not `grep`/`sed`/`cat` via Bash. The Write tool refuses to
   overwrite a file you haven't Read in this session, and a /clear
   resets that tracking; if you only inspected `STATUS.md` via Bash
   you will later hit `Error writing file` when updating it. Prefer
   `Edit` over `Write` for `STATUS.md` regardless — it always exists
   post-target and you usually only change the focus/last-commit
   lines.

6. **Look for the `Top-3 next subgoals` section at the top of the
   ROADMAP issue body.** That's the orchestrator's pre-picked
   priority list, written at the end of the previous session. Use it
   directly unless an open issue has clearly more recent context
   (e.g. a critic review that landed since).

7. Synthesize the current state in 2–3 sentences. Update STATUS.md
   (current focus + last commit + blocker). **Note the current
   `git rev-parse --short HEAD`** — this is the start-of-session SHA
   for the TIMELINE block's commit-range compare link (session-end
   Step 4). Commit: `status: session resume — <one-line
   current-focus>` and push.

8. **Sanity-build.** `(cd formal && lake build)` — one command
   builds the whole Lake project incrementally. If anything regresses
   (a previously-verified module now fails), fix that first before
   adding new content. (On a freshly-`/target`-bootstrapped clone,
   `formal/<DisplayName>/` will be empty — `lake build` will exit 0
   trivially.)

# Scope tier and time-to-first-deliverable

Read the `## Scope tier:` line at the top of the ROADMAP body (set by
`/target`). It governs how this session behaves:

- **T1 (known/textbook).** Rip straight through the literal-ask
  checkboxes: novelty gate → import wrapper or short formalization →
  promote → rebuild PDF → close. **Skip the standing librarian survey**
  (the novelty gate's loogle + leansearch is enough for settled
  material); do **not** open sub-vectors for a "cite it" / "comment on"
  checkbox. A T1 clone should reach `N/N` and halt in a **single short
  session (minutes)** — if you find yourself reaching for a multi-hour
  formalization on a T1 clone, the scope-fidelity gate has been
  violated. (`/target` may already have one-shotted the asks that were
  in Mathlib; just verify, finish any remainder, and halt.)
- **T2 / T3.** The full machinery applies — librarian survey,
  `research-state.md`, domain agents, reductions, partial progress.

**Time-to-first-deliverable (every tier).** Optimize for a readable
deliverable *early*. The first substantive thing a session produces
should be a *current, compiled* `proof.pdf` reflecting present state
(build it early even if it only shows the problem statement and first
sketch), and prioritize landing the **first verified result fast** over
breadth. A reader who opens `proof.pdf` should see real content within
the first few commits — not after a multi-hour wait. The PDF is rebuilt
and committed after every promotion and substantive manuscript change
(see "Manuscript PDF" and the verify cycle), so the landing-page
deliverable keeps rewriting itself as the session discovers.

# Subgoal priority (when picking from the Issues queue)

When multiple issues are open, prefer subgoals in this order:

0. **Novelty gate.** Run the gate before *any* fresh formalization.
   If the result is already in Mathlib, the work collapses to an
   import wrapper — closes the issue in one cycle.

1. **Promote an existing `[sketch]` to `[verified]`.** Sketches
   already carry the math content; formalizing them is the cheapest
   verified artifact a session can produce. (Look for issues
   labelled `sketch + open`.)
2. **Wiring / corollary work.** Combining existing verified pieces
   into a new theorem. Cheap and high-leverage.
3. **New sketch from a sketch-ready target.** A target that has a
   `findings/lit-*` survey and a sketch ready but no formal file.
4. **Numerical extensions** — **CAVEAT: not progress below published
   SOTA.** Extending a `[numerical: N]` claim to `[numerical: 10N]`
   is cheap and mechanical, but reproducing a range already covered
   by the published literature does not move the open question.
   Pursue numerical work only when it (a) tests a *new* conjecture
   not in the literature, (b) feeds directly into an analytic /
   combinatorial estimate the next session will use, or (c) is the
   last-resort tie-breaker between two equally uninteresting options.
   **Do not treat as a fallback to fill a session.** See CLAUDE.md
   §"Definition of progress".
5. **Fresh literature survey.** Only if the topic is genuinely
   uncovered (no `findings/lit-*-*.md`) and is needed by an upcoming
   target. **Anti-pattern**: dispatching `librarian` repeatedly
   without acting on the findings.
6. **Speculative new investigation.** Open a sketch, write a
   finding, but expect this to not produce a verified artifact this
   session.

Within each tier, prefer the issue with the most recent comment (it
has the freshest context) or the one referenced by the ROADMAP's
`Top-3 next` section.

# Verify cycle (the named pattern referenced by the work loop)

For any subgoal that ends in a `[verified]` Lean promotion, run this
exact sequence:

```
0. Run the novelty gate. If Mathlib already has it, skip to step 3
   with a thin import wrapper.
1. Pick subgoal from Issues, comment "Working on this — session DATE".
2. Sketch in proof.tex tagged [sketch] (optional if subgoal already
   has one).
3. Dispatch formalist → produces formal/<DisplayName>/Foo.lean that
   compiles cleanly under `lake build`.
4. Independently re-run `(cd formal && lake build)` + capture the
   `#print axioms ThmName` output verbatim. Do NOT trust the
   sub-agent's report alone.
5. Dispatch critic adversarially → writes findings/review-<topic>-DATE.md.
   (High-stakes promotion — the `novel` tag, the main theorem, or a
   result that appears to brush a published wall — runs the multi-lens
   critic panel instead; see "Workflow orchestration", Pattern A.)
6. If critic PASS: promote in proof.tex per the "Manuscript style"
   section below — update the `% [sketch]` LaTeX comment to
   `% [verified: formal/<DisplayName>/Foo.lean] [<novelty>]` (the
   trailing novelty token from the gate: `novel` / `formalization` /
   `known-cited`), replace any `\begin{sketch}` block with a
   publication-quality `\begin{proof}`, pull out auxiliary sub-claims
   as standalone `\begin{lemma}` blocks with their own `\begin{proof}`
   blocks, add a discreet footnote on the top-level theorem pointing
   to the `.lean` file, and update (or create) the `Formal
   verification` appendix. Do NOT add a "Critic review: PASS" remark
   inside the manuscript and do NOT update a "Revision history"
   section — that metadata lives in the closed issue, in
   `findings/review-<topic>-<date>.md`, and in the git log.
7. Commit `formalize: …` (step 3), `review: critic PASS on …`
   (step 5), `promote: … [verified]` (step 6). Then **rebuild and
   commit the PDF** — `(cd manuscript && tectonic -X compile
   proof.tex)`, `git add manuscript/proof.pdf`, `pdf: rebuild after
   <promotion>`. Four commits. Push each. (The PDF rebuild on every
   promotion is what keeps the landing-page deliverable fresh — see
   "Manuscript PDF".)
8. Comment on the issue with the verifying commit SHA, drop labels
   `open`/`sketch`, add label `verified` **and** the novelty label
   (`novel` / `formalization` / `known-cited`). Use `Closes #N` in
   the commit body so push auto-closes.
9. Update STATUS.md current-focus line. Commit if changed.
```

For a `[numerical: range]` subgoal, replace steps 3–6 with: dispatch
`computationalist` → produce script + transcript + summary; verify 0
failures; promote in proof.tex by changing the `% [sketch]` LaTeX
comment to `% [numerical: <range>]` and stating the result as a
proposition with a brief "by direct computation; see
`formal/numerics/<script>.py`" proof body. No critic step is
required for a clean computational result, but if the result is
surprising ask `critic` to sanity-check the methodology.

# External-solver bridge (SAT/SMT → Lean)

When a subgoal reduces to a finite SAT/SMT search whose witness or
unsat certificate needs to feed a machine-checked Lean theorem:

1. `complexity-theorist` designs the CNF/SMT encoding and documents
   the variable mapping.
2. `computationalist` runs the solver and saves the LRAT certificate
   (`cadical --lrat`, `kissat --proof`) to
   `formal/numerics/<topic>.lrat`. Commit the certificate.
3. `formalist` writes a `.lean` file that imports the certificate
   and applies a Lean LRAT verifier. The Lean LRAT verifier is **not
   built as part of this template** — the first clone that needs SAT
   verification ports a verifier (adapting an existing Lean
   LRAT-checker package, or writing one); subsequent clones reuse.

Until the verifier exists in the project, SAT-derived results sit at
`[numerical: <range>]`, not `[verified]`. Document this honestly in
the manuscript and in the issue.

# Manuscript style (publication-quality, not project log)

`manuscript/proof.tex` is the publication-quality paper for this
clone. It is **not** a project log. A reader who opens
`manuscript/proof.pdf` on GitHub should see something that reads like
an arXiv preprint: title, abstract, introduction with the problem and
context, a main theorem with a proof, auxiliary lemmas with their own
proofs, optional discussion, optional `Formal verification` appendix.
Project scaffolding lives elsewhere:

| Scaffolding | Lives in |
|---|---|
| Attack vectors / roadmap | ROADMAP GitHub issue |
| Session log, current focus | `STATUS.md` |
| Inter-agent notes, reviews | `findings/` |
| Revision history | `git log` |

This separation is load-bearing. Mixing scaffolding into the
manuscript was the standing failure mode that motivated this section.

## Required structure

A manuscript that has reached at least one verified result should
have roughly this shape:

```latex
\title{<problem display name>}
\author{<GIT_USER_NAME>}
\date{\today}

\begin{document}
\maketitle

\begin{abstract}
<paper-style abstract, 4-6 sentences: what problem, what is the main
result, one-sentence summary of the proof strategy.>
\end{abstract}

\section{Introduction}
\label{sec:intro}

<problem statement, context / motivation, brief announcement of the
main result. End with a one-line outline of the paper if more than
two sections of content follow.>

\section{Main result}
\label{sec:main}

% [verified: formal/<DisplayName>/Main.lean]
\begin{theorem}\label{thm:main}
<formal statement>.\footnote{Formally verified in Lean 4 + Mathlib
(commit \texttt{<short-sha>}); see
\texttt{formal/<DisplayName>/Main.lean}.}
\end{theorem}

The proof relies on two lemmas, established below.

% [verified: formal/<DisplayName>/Main.lean]
\begin{lemma}\label{lem:aux1}
<auxiliary statement 1>.
\end{lemma}
\begin{proof}
<flowing mathematical prose>.
\end{proof}

% [verified: formal/<DisplayName>/Main.lean]
\begin{lemma}\label{lem:aux2}
<auxiliary statement 2>.
\end{lemma}
\begin{proof}
<flowing mathematical prose>.
\end{proof}

\begin{proof}[Proof of Theorem~\ref{thm:main}]
<flowing mathematical prose combining the lemmas>.
\end{proof}

% Optional, once content warrants:
\section{Discussion}
or
\section{Concluding remarks}

% Optional, once at least one [verified] result exists:
\appendix
\section{Formal verification}
\label{sec:formal}

The result(s) above have been formally verified in Lean 4 against
Mathlib (commit \texttt{<short-sha>}). The proof scripts live under
\texttt{formal/<DisplayName>/} and compile under \texttt{lake build};
\texttt{\#print axioms} on each theorem reports only the Lean
foundational axioms (\texttt{propext}, \texttt{Classical.choice},
\texttt{Quot.sound}).

\begin{itemize}
  \item Theorem~\ref{thm:main}: \texttt{formal/<DisplayName>/Main.lean}
  \item (further entries as more results are verified)
\end{itemize>

\end{document}
```

The Mathlib commit SHA is read from `formal/lake-manifest.json` and
baked into the footnote / appendix on each promotion, so a reader
can reproduce the build exactly.

## Banned from the manuscript

- `\section{Initial attack vectors}` / `\section{Roadmap}` / any
  attack-vector list — those live in the ROADMAP issue.
- `\section{Revision history}` — git log is the history. If the
  template had one (older clones do), remove it on next promotion.
- `\section{Status}` / `\section{Current focus}` — `STATUS.md`.
- `\section{Verified results}` / `\section{Sketches}` — pre-emptive
  container sections. Theorems live where the mathematical
  exposition puts them; their tag-comments (below) say what's
  verified vs. sketched.
- `\begin{sketch}...\end{sketch}` blocks for results that have
  already been formally verified — on promotion, replace with
  `\begin{proof}...\end{proof}`.
- Inline `\texttt{[verified: formal/X.lean]}` text in section or
  theorem headings — verification status is tracked by the LaTeX
  comment above the theorem (machine-readable, invisible in PDF)
  plus an optional footnote on the statement.
- "Step 1 / Step 2 / Step 3" labels inside proofs (or
  `\emph{Step 1: ...}`) — use natural mathematical prose: "We
  proceed by induction on $n$. The base case $n=1$ is immediate
  from … For the inductive step, …". Step-labels read as
  scratchpad, not paper.
- Project-commentary remarks inside the manuscript — "Critic
  adversarial review of YYYY-MM-DD: PASS", "Pending Lean
  formalization", "See issue #17" — that metadata belongs in the
  closed issue and in `findings/review-*.md`, not in the paper.

## Machine-readable verification tags

Carry a single-line LaTeX comment immediately above every theorem,
lemma, and proposition. The comment is invisible in the rendered PDF
but lets future sessions (and `grep`) determine the verification
status of each statement:

```latex
% [verified: formal/<DisplayName>/Divergence.lean] [novel]
\begin{theorem}\label{thm:divergence}
The series $\sum_{n=1}^\infty p_n^{\,n}/n!$ diverges.\footnote{Formally
verified in Lean 4 + Mathlib; see
\texttt{formal/<DisplayName>/Divergence.lean}.}
\end{theorem}
\begin{proof}
<flowing prose proof>.
\end{proof}
```

Valid tags:

- `% [sketch]` — informal argument present (or no argument yet);
  not yet formalized.
- `% [verified: formal/<DisplayName>/File.lean] [<novelty>]` —
  formally verified; `lake build` succeeds, `#print axioms` is clean,
  `critic` review PASS. The trailing `[<novelty>]` token is one of
  `[novel]` / `[formalization]` / `[known-cited]` (see the novelty
  gate). The existing `grep -nE '%\s*\[verified: formal/'` still
  matches — the token is a suffix.
- `% [conditional on <Hypothesis>]` — depends on a named open
  hypothesis (RH, GRH, etc.).
- `% [numerical: <range>]` — finite-range computational
  verification.
- `% [heuristic]` — empirical pattern only.

Attach the discreet footnote linking to the `.lean` file **only on
the top-level statement** the reader encounters (typically the main
theorem). Auxiliary lemmas in the same `.lean` file are listed in
the Formal verification appendix, not footnoted individually, so the
paper does not become a forest of footnotes.

## Promotion: rewriting sketch → publication-style proof

When critic PASS lets you promote a `[sketch]` to
`[verified: formal/<DisplayName>/Foo.lean]`, the manuscript edit is
more than a tag flip:

1. **Update the LaTeX comment** from `% [sketch]` to
   `% [verified: formal/<DisplayName>/Foo.lean] [<novelty>]` (novelty
   token from the gate: `novel` / `formalization` / `known-cited`).
2. **Rewrite the proof in publication style.** If the sketch was
   informal ("Step 1 …", "Step 2 …", `\emph{Step 1: ...}`), replace
   with a `\begin{proof}…\end{proof}` block written as flowing
   mathematical prose. Cut any project-side commentary.
3. **Pull out auxiliary lemmas.** Any sub-claim used in the main
   proof that is independently meaningful becomes a standalone
   `\begin{lemma}` with its own `\begin{proof}` block, stated
   before the main theorem so the main proof can cite it via
   `Lemma~\ref{lem:foo}`. Each lemma gets its own
   `% [verified: …]` comment (often the same `.lean` file as the
   main theorem).
4. **Add the theorem footnote.** Top-level theorem only — a single
   `\footnote{Formally verified in Lean 4 + Mathlib (commit
   \texttt{<short-sha>}); see
   \texttt{formal/<DisplayName>/Foo.lean}.}` on the statement.
5. **Update or create the abstract.** If this is the first verified
   result, replace the placeholder abstract with a paper-style
   abstract (4-6 sentences: problem, main result, proof strategy in
   one sentence). On later promotions, extend the abstract if the
   new result widens the scope; do not let it become a list.
6. **Update or create the Formal verification appendix.** If this
   is the first `[verified: …]` claim, add the `\appendix` block
   per the structure template above. On later promotions, add an
   itemize entry pointing to the new theorem and `.lean` file.
7. **Tighten the introduction.** Once the main result is verified,
   the introduction's closing sentence ("currently in preparation"
   from /target) should be replaced with a real one-line
   announcement of the result and a forward-reference to
   Theorem~\ref{thm:main}.
8. **Do not** add a "Critic review YYYY-MM-DD: PASS" remark in the
   manuscript and **do not** touch a "Revision history" section
   (the manuscript has none; if you find one in an old clone,
   remove it as part of this promotion).

After the rewrite, the manuscript section reads as if it had been
written for publication from the start. A reader who never sees the
issue tracker, STATUS.md, or git history should still understand the
mathematics in full.

# Work loop

**Narration discipline (do this throughout the loop).** The user
runs `/loop /solve` unattended and reads the live turn to know what's
happening — the recurring complaint was having to ask "what's going
on" during long silent stretches while a subagent ran. So: **before
every `Agent` dispatch and after every batch returns, emit one concise
TEXT line** (plain output, not a tool call):
- On dispatch: `→ [<agent>] <one-clause task> (subgoal #<N>)`
- On return: `← [<agent>] <one-clause outcome> → next: <what happens with the result>`

Example: `→ [formalist] Containment.lean (subgoal #14)` then
`← [formalist] compiles sorry-free → next: #print axioms + critic`.
One sentence each; don't narrate *while* a subagent runs (the dispatch
line already says what's awaited). This narration is for the live
terminal only — the durable per-session record is the README TIMELINE
block (session end). Do **not** post recurring progress comments to
GitHub issues; that would clutter the tickets the sweep is closing.

Repeat the following cycle until you decide the session is at a
clean stopping point:

a. **Pick a subgoal** per the priority rubric above. From the open
   Issues queue (`gh issue list --state open --label open`), pick
   one. State it precisely in one sentence. If you pick an issue,
   claim it by commenting `Working on this — session 2026-MM-DD`.

   **Before picking, re-check the target-reached condition** (Rule
   10): if the queue is empty *and* the ROADMAP shows `N/N closed`,
   exit the work loop and proceed to the session-end protocol — do
   **not** auto-open a new vector. Auto-opening is permitted *only*
   while the ROADMAP meter is `X/N` for `X < N`, and only when the
   new vector visibly advances the originally stated target (per
   the "Auto-opening" subsection of Rule 10).

b. **Novelty gate.** Run loogle + leansearch (and librarian if
   neither hits). If Mathlib has the result, the subgoal collapses
   to an import wrapper.

c. **Literature check.** *(Skip on T1 clones — the novelty gate
   suffices for settled material; a full survey on T1 is the 5.7h
   failure mode.)* If the subgoal is novel or it has been > a few
   sessions since this topic was last surveyed, dispatch `librarian`
   to ground it in recent literature. (For a broad T2/T3 survey, the
   parallel **research sweep** — § Workflow orchestration, Pattern B —
   fans the sources out and merges them, faster than the serial
   single-librarian pass.) The librarian reads existing
   findings first (no re-surveying), runs a WebSearch sweep, saves the
   survey under `findings/lit-<topic>-<YYYY-MM-DD>.md`, and **refreshes
   `findings/research-state.md`** (the single consolidated known-vs-open
   picture + per-checkbox novelty hint). Add an entry to
   `findings/INDEX.md`, and cite `research-state.md` from the ROADMAP
   body so the research picture is visible from the dashboard. Comment
   on the related open issue with the findings path; or if the survey
   opens a brand-new target (subject to the scope-fidelity gate), open
   a new issue.

d. **Sketch.** Dispatch the appropriate domain agent (`analyst`,
   `sieve-theorist`, `combinatorialist`, `geometer`, `algebraist`,
   `topologist`, `probabilist`, `complexity-theorist`) — or two in
   parallel if independent — to produce an informal sketch. Add the
   sketch to `manuscript/proof.tex` tagged `[sketch]`.

e. **Formalize / compute.** Two cases:
   - If the claim is a logical statement (lemma, definition,
     equivalence), dispatch `formalist` to produce a `.lean` file
     under `formal/<DisplayName>/`. It must compile (a `sorry` is
     allowed, but the manuscript entry stays `[sketch]` until the
     `sorry` is gone).
   - If the claim is a finite computation, dispatch
     `computationalist` to produce a script under
     `formal/numerics/` plus a transcript. The resulting manuscript
     entry is `[numerical: <range>]`, not `[verified: …]`.

f. **Review.** Dispatch `critic` to read the sketch + the `.lean`
   file. Address concerns concretely. If the review finds a fatal
   gap, downgrade the manuscript tag and document in
   `findings/review-*.md`. For a high-stakes promotion (the `novel`
   tag, the main theorem, or a wall-brusher), run the multi-lens
   **critic panel** instead — § Workflow orchestration, Pattern A.

g. **Promote.** If steps (e) and (f) succeed:
   - Change the manuscript tag from `[sketch]` to `[verified:
     formal/<DisplayName>/File.lean] [<novelty>]` (novelty token from
     the gate).
   - Close the issue with a comment pointing to the commit SHA(s).
   - Update labels on the issue: drop `open`/`sketch`, add
     `verified` or `numerical`, plus the novelty label.

h. **Commit + push, then rebuild the PDF.** See "Commit format"
   below. Reference the issue number in the commit body (e.g.,
   `Closes #17` to auto-close on push). After a promotion (or any
   substantive manuscript change), rebuild and commit
   `manuscript/proof.pdf` (`pdf: rebuild after <reason>`) so the
   landing-page deliverable stays current — see "Manuscript PDF".

i. **Update STATUS.md** with the new current-focus line (one line
   only) and proceed to (a). Roadmap-level changes go to the ROADMAP
   issue.

When two subgoals are independent, dispatch their domain agents in
parallel (one message, multiple `Agent` tool uses). See the
dedicated parallelism section below.

# Parallelism and inter-agent communication

The team is designed to work concurrently with zero user input.

**Concurrency model.** Multiple subagents dispatched in a single
message run **in parallel**, each in its own context window with its
own tool calls. The orchestrator (this skill, /solve) collects their
returns and dispatches the next batch.

**Communication channels.** Agents do not share live state. They
communicate **asynchronously through files and Issues**:
- `findings/` — markdown notes, naming convention in
  `findings/INDEX.md`.
- `manuscript/proof.tex` — sketches and verified claims.
- `formal/<DisplayName>/*.lean`, `formal/numerics/*` — formal
  sources and scripts.
- GitHub Issues — the live work queue (created/closed/commented by
  /solve, not by sub-agents).
- `STATUS.md` — one-line live pointer.
- git log — durable history of decisions and breakthroughs.

When `librarian` drops a survey at `findings/lit-X-DATE.md`, the
next agent that needs that topic reads it instead of re-searching.
Sub-agents should **not** themselves call `gh issue`; they leave
outputs in files and report back to /solve, which then updates
Issues.

**Safe parallel work.** When dispatching parallel agents, ensure no
two agents write to the same file:
- `librarian` on topic A + `formalist` on Lemma B (different files) ✓
- `analyst` and `sieve-theorist` exploring different sub-obligations ✓
- `computationalist` writing `formal/numerics/scriptA.py` while
  `formalist` writes `formal/<DisplayName>/LemmaB.lean` ✓

**Unsafe (do not parallelize):**
- Two agents both editing `manuscript/proof.tex`.
- Two agents both editing `STATUS.md`.
- Two agents both writing to the same `.lean` file.
- Two agents both running `gh issue …`.

After parallel agents return, /solve serially updates the shared
files (`proof.tex`, `STATUS.md`, `findings/INDEX.md`), runs the
appropriate `gh issue` commands, then commits and pushes.

**Zero user input.** No agent — and not the /solve orchestrator —
ever calls `AskUserQuestion`. If the team needs a decision that
can't be derived from the repo, /solve picks the option most
consistent with the existing precedent (read prior commits, prior
findings, prior `proof.tex` style), records the choice in
`findings/decision-<topic>-<date>.md`, and continues.

# Workflow orchestration (optional power-tool for fan-out)

The "dispatch N `Agent` calls in one message" model above is the
default. For two recurring **fan-out** shapes the **`Workflow` tool** is
a better fit: it runs a deterministic script that spawns a fleet of
subagents, collects schema-validated returns, and (via `agentType`)
reuses the existing agent definitions verbatim. The Workflow tool
requires explicit opt-in; **these two patterns, invoked from this skill,
are that opt-in.** Outside them, prefer plain `Agent` dispatch — do not
reach for a Workflow for a single critic pass or a one-shot lookup, and
respect the same cost discipline as the 30-min anti-stagnation cap (a
panel is ~5 agents, a sweep ~6; don't fan out a trivial import wrapper).

**The orchestrator stays the spine.** A Workflow returns *data*; /solve
still owns everything that must stay legible and incremental:
- the `→`/`←` narration lines — emit `→ [workflow:critic-panel] …` on
  launch and `← [workflow:critic-panel] 5 lenses, 0 refutations` on
  return (the `/workflows` view shows live per-agent progress; the turn
  text still needs the two lines);
- the independent `lake build` + `#print axioms` re-check (never trust
  an agent's self-report — verify-cycle Rule 4);
- writing the shared files (`proof.tex`, `findings/*`,
  `research-state.md`), running `gh issue`, committing per work unit,
  and rebuilding the PDF.

Agents inside a Workflow obey the same "no shared-file writes, no `gh`"
rule as any sub-agent (§ Parallelism, "Unsafe"). The skeletons below are
*adapt-me* templates — persist via the Workflow tool's `scriptPath`,
tweak the lens/angle text per problem.

## Pattern A — adversarial critic panel (harden a high-stakes promotion)

The default verify cycle runs **one** critic. For a **high-stakes
promotion** — anything proposing the `novel` tag, the main theorem, or a
result that appears to brush a published wall — fan `critic` out across
its own heuristics (§ critic.md), **one lens per agent**, and let any
credible refutation block the promotion. Five distinct lenses catch what
five identical passes would miss.

```js
export const meta = {
  name: 'critic-panel',
  description: 'Adversarial multi-lens critic panel for one [verified] promotion candidate',
  phases: [{ title: 'Refute' }],
}
// One lens per critic, drawn from critic.md's heuristic list:
const LENSES = [
  'Hidden dependence on a famous open hypothesis (RH/GRH/EH/GEH/Hardy–Littlewood, a density hypothesis, or any conjecture the ROADMAP lists as a TARGET) or the parity barrier — smuggled in via "standard estimates".',
  'Stronger-than-target: does the argument as stated prove something strictly stronger than the target? If so it almost certainly has a bug — find the unstated assumption.',
  'Prose elisions and arithmetic: expand every "WLOG"/"clearly"/"standard"; check uniformity in every parameter (N, modulus q, residue a); recompute the main term for sign / scale / off-by-one errors.',
  'Lean acceptance gate: #print axioms lists ONLY propext / Classical.choice / Quot.sound (no sorryAx, no Lean.ofReduceBool, no project-local axiom); the Lean statement matches the manuscript claim; no [conditional]→[verified] laundering.',
  'Novelty honesty: is the proposed `novel` tag real, or a known result / Mathlib cousin dressed up? Cross-check research-state.md + a quick loogle/leansearch.',
]
const VERDICT = { type: 'object', additionalProperties: false,
  properties: { refuted: { type: 'boolean' }, reason: { type: 'string' } },
  required: ['refuted', 'reason'] }
// args = { claim: '<manuscript statement>', lean: '<.lean path + verbatim #print axioms output>' }
const votes = await parallel(LENSES.map((lens, i) => () =>
  agent(`You are critic. Review ONLY through this lens:\n${lens}\n\n`
      + `Candidate: ${args.claim}\nLean: ${args.lean}\n\n`
      + `Default to refuted=true if you cannot rule the failure out.`,
    { agentType: 'critic', label: `critic:lens${i + 1}`, phase: 'Refute', schema: VERDICT })))
const refutations = votes.filter(Boolean).filter(v => v.refuted)
return { promote: refutations.length === 0, refutations }
```

On return: `promote === true` (zero refutations) → proceed with the
promotion. Any refutation → read each `reason`, send it back to
`formalist` (Lean defect) or downgrade the manuscript tag, and record
the panel verdict in `findings/review-<topic>-<date>.md` exactly as a
single-critic review would. The panel **replaces** verify-cycle step 5 /
work-loop step (f) for high-stakes promotions; routine import wrappers
keep the single pass.

## Pattern B — parallel research sweep (speed up a librarian survey)

The default literature check runs **one** librarian serially through all
its sources. For a genuine survey (**T2/T3 only — never T1**), fan the
search across independent angles in parallel, then a final librarian
merges them into the one known-vs-open picture. Wall-clock drops to the
slowest single angle instead of the sum.

```js
export const meta = {
  name: 'research-sweep',
  description: 'Parallel multi-angle literature sweep, merged into one research picture',
  phases: [{ title: 'Sweep' }, { title: 'Synthesize' }],
}
const ANGLES = [
  'Mathlib novelty gate: loogle (type-shaped) + leansearch (English). Already formalized? Record exact module paths.',
  'arXiv, last 3–5 years, primary category: the SOTA bound + the 3 closest predecessors, with arXiv ids and theorem #s.',
  'MathOverflow + Tao blog + Polymath + erdosproblems + AIM: known obstructions, folklore, partial results.',
  'Google Scholar: most-cited landmarks and survey / expository articles on the target.',
  'Reservoir + recent arXiv-with-Lean-source: existing partial formalizations to reuse (package + git URL + SHA).',
]
const HITS = { type: 'object', additionalProperties: false, required: ['hits'],
  properties: { hits: { type: 'array', items: { type: 'object', additionalProperties: false,
    required: ['ref', 'gives', 'doesnt'],
    properties: { ref: { type: 'string' }, gives: { type: 'string' }, doesnt: { type: 'string' } } } } } }
// args = { topic: '<subgoal in one sentence>' }
const sweeps = await parallel(ANGLES.map((a, i) => () =>
  agent(`You are librarian. Cover ONLY this angle:\n${a}\n\nTopic: ${args.topic}\n\n`
      + `Return precise pointers (arXiv id / theorem # / Mathlib module path) with what each DOES and DOES NOT give.`,
    { agentType: 'librarian', label: `sweep:angle${i + 1}`, phase: 'Sweep', schema: HITS })))
const merged = await agent(
  `You are librarian. Merge these per-angle findings into ONE known-vs-open picture: name the closest `
  + `predecessor and the strengthening that would close the gap, propose 3 candidate attack vectors, and give a `
  + `per-checkbox novelty hint (known-cited / formalization / candidate-novel).\n\n`
  + JSON.stringify(sweeps.filter(Boolean)),
  { agentType: 'librarian', phase: 'Synthesize' })
return merged
```

On return: /solve writes `findings/lit-<topic>-<date>.md`, refreshes
`findings/research-state.md` from `merged`, adds the `findings/INDEX.md`
entry, cites `research-state.md` from the ROADMAP, and commits — the same
artifacts the serial path produces, gathered faster. The sub-agents wrote
nothing to disk; the orchestrator owns the files.

# Work-unit definitions (any one of these = commit)

- A new Lean-verified theorem compiled by `lake build` (no `sorry`).
- A new theorem formalized with `sorry` gaps and a corresponding
  sketch in `manuscript/proof.tex`.
- A new finding file under `findings/` (literature, heuristic, dead
  end, open question, decision, review).
- A new section or paragraph added to `manuscript/proof.tex`.
- A new or updated GitHub Issue.
- A rebuild of `manuscript/proof.pdf` (after every promotion and
  substantive manuscript change, and at session end).

Don't bundle multiple work units into one commit; small commits make
the GitHub log readable.

# Commit format

```
<verb>: <short specific subject, < 60 chars>

<optional body, wrapped at 72 cols. Cite arXiv ids, theorem numbers,
and issue numbers (e.g., "Closes #17" to auto-close on push).>
```

**Commit authorship is non-negotiable.** Every commit is authored as
**<GIT_USER_NAME> <<GIT_USER_EMAIL>>** (the repo-local git config
already set by `make init`). **Never** include:
- `Co-Authored-By: Claude …` trailers
- `Generated with Claude Code` footers
- any mention of Claude, Anthropic, or the agent runtime

The repo is presented as the user's own work. The tooling is
invisible.

Verbs: `formalize`, `sketch`, `survey`, `verify`, `promote`,
`revise`, `deadend`, `status`, `chore`, `pdf`, `vector`, `pivot`.

Examples (problem-agnostic):
- `formalize: containment criterion — formal/<DisplayName>/Containment.lean`
- `sketch: <subgoal> in proof.tex section <N>`
- `survey: <topic> bounds <year-range>`
- `deadend: Selberg sieve hits parity barrier on <set>`
- `status: switch focus to <new-subgoal>`
- `pdf: rebuild proof.pdf after <change>`

# Manuscript PDF (rebuilt per promotion + at session end)

`manuscript/proof.pdf` is **committed** to the repo so the user can
read the rendered manuscript directly on GitHub (click the file in
the file listing to open the inline PDF viewer).

**Rebuild and commit it after every promotion and every substantive
manuscript change** — not only at session end. This is the
"rewrite-as-it-discovers" cadence: a reader who opens `proof.pdf`
should see real content within the first few commits, and watch it
grow, rather than wait for a multi-hour session to finish. (Skip the
rebuild for status/chore/findings-only commits that don't touch
`proof.tex`.) The extra binary churn in git history is an accepted
trade for visible, early, continuously-fresh deliverables.

Build:
```sh
(cd manuscript && tectonic -X compile proof.tex)
```

`tectonic` auto-downloads any missing LaTeX packages on first run;
no admin texlive setup needed. Install with `brew install tectonic`
if not present.

**Quirks**:
- LaTeX environment options that contain literal `[` or `]` must be
  brace-protected: `\begin{proposition}[{Numerical bound on $[8,
  10^6]$}]`, not `\begin{proposition}[Numerical bound on $[8,
  10^6]$]`.

After a successful build, commit:
```sh
git add manuscript/proof.pdf
git commit -m "pdf: rebuild after <reason>"
```

# Session-end protocol

You decide when to stop. A good stopping point is after a verified
promotion, after exhausting your immediate plan, or when context is
getting full. On stop:

1. **Rebuild the PDF**: `(cd manuscript && tectonic -X compile
   proof.tex)` (subshell so cwd doesn't leak; the next `git add
   manuscript/proof.pdf` from the project root would fail
   otherwise). Commit `manuscript/proof.pdf` separately as
   `pdf: ...`.

1a. **Ticket sweep — enforce the one-open-issue invariant.** Run this
   *before* the ROADMAP/README refresh so the meters reflect the
   swept state. **Invariant: at rest, exactly one issue is open — the
   ROADMAP.** Find every other open issue and dispose of it:

   ```sh
   ROADMAP_NUM=$(gh issue list --label roadmap --json number --jq '.[0].number')
   gh issue list --state open --json number,title,labels \
     --jq ".[] | select(.number != $ROADMAP_NUM) | .number"
   ```

   For each returned issue:
   - **Verified/closed work** → if not already closed by a `Closes #N`
     commit, close it, drop `open`/`sketch`, add `verified` (and the
     novelty label).
   - **Exhausted** (hit the ~30-min anti-stagnation cap, or every
     seeded search line tried) → close, `--remove-label open
     --add-label deadend`, comment citing the `findings/deadend-*.md`.
   - **Deferred** (not dead, just not now) → close, `--add-label
     deprioritized`, comment "revisit via `/vector add`."
     (Reuse `/vector retire`'s keyword heuristic: blocked / impossible
     / barrier / dead-end wording → `deadend`, else `deprioritized`.)

   **Sole carry-over exception:** an issue named in the freshly-written
   `## Top-3 next subgoals` may stay open *between* sessions (it's the
   next session's first pick). At a **halt** (`N/N` or exhaustion),
   even those close — leaving only the ROADMAP. **Never close the
   ROADMAP** (the user closes it manually). Then assert the invariant:

   ```sh
   echo "open issues at session end: $(gh issue list --state open --json number --jq 'length') (target at rest: 1 = ROADMAP)"
   ```

2. **Update the ROADMAP issue**. Three parts:
   - **Body checkboxes**: GitHub auto-ticks `- [ ] #N <title>`
     entries when issue #N closes (via the tracked-in feature).
     Re-read the issue body and confirm the auto-ticks reflect
     actual state; manually correct any drift.
   - **`## Progress: N/M closed`** line: recount the checkboxes and
     update.
   - **`## Top-3 next subgoals`** section at the top: rewrite it
     with the three highest-priority issues for the next session,
     each as `- [#N issue title](../issues/N) — <one-line why>`.
     This is what the next session reads at resume step 6; keep it
     fresh.
3. **Update STATUS.md** (one-line `Current focus`; one-liner `Last
   commit`).
4. **Refresh README.md status blocks.** The per-clone README has
   **five** HTML-comment-bounded sections that this protocol keeps in
   sync with the manuscript and the Issues tab (STATUS, VERIFIED,
   OPEN, ASSESSMENT, TIMELINE). Rewrite the content between each
   `<!-- BEGIN X -->` / `<!-- END X -->` pair *without moving or
   removing the marker lines themselves*.

   - **STATUS block** (`<!-- BEGIN STATUS --> ... <!-- END STATUS -->`)
     — five bullets:
     * `Last session`: today's UTC date (`date -u +%F`).
     * `Roadmap`: `[#<N>](../../issues/<N>)` where `<N>` is the
       ROADMAP issue (normally `#1`; use `gh issue list --label
       roadmap --json number --jq '.[0].number'` to confirm).
     * `Verified results`: `X / Y`, where `X` is the count of
       `[verified: ...]` tags in `manuscript/proof.tex` and `Y`
       is the total sub-lemma checkbox count in the ROADMAP body
       (parse via `gh issue view <N> --json body --jq '.body' |
       grep -c '^- \[[ x]\] #'`).
     * `Current focus`: identical to the focus line you just
       wrote into `STATUS.md` (single source of truth).
     * `Manuscript`: keep the
       ``[`proof.pdf`](manuscript/proof.pdf)`` link.

   - **VERIFIED block** (`<!-- BEGIN VERIFIED --> ... <!-- END VERIFIED -->`)
     — a markdown table, one row per verified theorem, with a
     **Novelty** column so a reader can tell a genuine contribution
     from a re-proof of a known fact at a glance:

     ```markdown
     | Theorem | Novelty | Lean file | Axiom check |
     |---|---|---|---|
     | <Name> | novel | [`formal/<DisplayName>/<File>.lean`](formal/<DisplayName>/<File>.lean) | clean |
     ```

     To populate: `grep -nE '%\s*\[verified: formal/' manuscript/proof.tex`
     gives the file references; the theorem name is the
     `\begin{theorem}[Name]` or
     `\begin{theorem}\label{thm:name-slug}` immediately below the
     tagged line. **Novelty** is the trailing `[novel]` /
     `[formalization]` / `[known-cited]` token on the same tag line;
     if a legacy comment lacks the token, default to `formalization`
     and flag it for backfill (so the table never breaks). "Axiom
     check" is `clean` when `#print axioms` in that file lists only
     `propext` / `Classical.choice` / `Quot.sound`; `native_decide`
     warrants the literal `native_decide (critic-approved)`. If no
     verified results yet, write `*No verified results yet.*` (single
     italics line, no table).

   - **OPEN block** (`<!-- BEGIN OPEN --> ... <!-- END OPEN -->`)
     — table of open obligations (cap at ~10 rows; if more,
     follow with `… and N more — see the [ROADMAP](../../issues/<N>).`):

     ```markdown
     | Obligation | Status | Issue |
     |---|---|---|
     | <issue title> | <derived from labels> | [#N](../../issues/N) |
     ```

     Source: `gh issue list --state open --label open --json
     number,title,labels --limit 20`. Status derives from labels:
     `sketch` → "sketch"; `numerical` → "numerical: \<range from
     issue body\>"; `conditional` → "conditional on
     \<hypothesis\>"; `survey` → "survey"; `review` →
     "under review"; nothing else → "open".

   - **ASSESSMENT block** (`<!-- BEGIN ASSESSMENT --> ... <!-- END ASSESSMENT -->`)
     — an **evidence ledger, NOT a forecast** (see CLAUDE.md
     §"Anti-defeatism" — this block is explicitly bound by the
     banned-framings rule). Four mechanically-derived bullets:
     1. **Target progress** — `X / N` rendered as a meter, e.g.
        `▰▰▰▱▱ 3 / 5 sub-lemmas verified` (`N` = ROADMAP checkbox
        count, `X` = closed-as-verified).
     2. **Verified vs sketched** — counts of `[verified]` /
        `[sketch]` / `[numerical]` / `[conditional]` tags in
        `proof.tex`.
     3. **Obstructions logged** — one line per
        `findings/deadend-*.md`, each `<concrete obstacle>
        (deadend-….md)`. This is the post-hoc evidence: only obstacles
        *hit during an attempt* appear here.
     4. **What the evidence shows** — ONE or TWO sentences strictly
        summarizing 1–3, **past-tense fact only**. Allowed: "All K
        seeded vectors were attempted; V reduced to a verified
        sub-lemma, D hit the obstruction(s) above, R remain in
        progress." **BANNED** (per CLAUDE.md): "best we can hope for",
        "this vector cannot settle X", "parity-barrier-blocked", any
        numeric probability, any sentence forecasting whether the
        target *will* resolve. State what HAS happened; never forecast
        what CAN happen. The "likelihood" signal the reader wants is
        delivered implicitly by the meter + verified ratio + the
        concrete walls hit — not by a verdict sentence.

   - **TIMELINE block** (`<!-- BEGIN TIMELINE --> ... <!-- END TIMELINE -->`)
     — a per-session log, **newest first**, one entry per `/solve`
     session, **capped at the 8 most recent** (when trimming, replace
     the tail with `… earlier sessions: see [git history](../../commits/main).`).
     Prepend this session's entry:

     ```markdown
     ### <YYYY-MM-DD> — <one-clause headline>
     - Did: <verified N / numerical N / surveys / dead-ends — what landed>
     - Issues: closed #…; opened #… (omit a half with nothing)
     - Commits: [`<first7>…<last7>`](../../compare/<first>...<last>)
     ```

     Build the compare range from the `HEAD` short-SHA captured at
     resume (Step 7 of the resume protocol committed
     `status: session resume`) and the current `HEAD`. The entry is
     written from the same facts as the end-of-session report (one
     source, two renderings — terminal report is ephemeral, this is
     the durable GitHub twin).

   **Refusing to refresh.** If any of the five sentinel pairs are
   missing from `README.md`, the clone was either not bootstrapped
   by `/target` or the README was hand-edited destructively.
   Append a one-line "skipped README refresh — sentinel(s) missing"
   to the session-end report; do **not** invent a new README from
   scratch (that's `/target`'s job, not `/solve`'s).

5. Commit: `status: session end — <one-line summary>` and push.
6. **Re-run the halt checks** (per Resume Protocol Step 0 and Rule 10).
   Two halt conditions:
   - **`N/N` reached** — this session's commits just ticked the final
     checkbox (`CLOSED == TOTAL`): output the `N/N` halt message
     exactly as in Step 0 of the resume protocol (actual issue number
     + `closed/total`), suggest `/polish`.
   - **Exhausted** — the queue is empty post-sweep and no
     scope-advancing vector can be auto-opened, even though
     `CLOSED < TOTAL`: write `findings/decision-exhausted-<date>.md`
     and output the **exhaustion** halt message (Rule 10, Halt
     condition 2), suggest `/vector`.

   In either case: confirm the sweep left exactly one open issue (the
   ROADMAP), do **not** call `ScheduleWakeup`, and skip the next
   bullet's "what's next" framing — the halt message covers it.
7. If the target is *not* reached, output a brief end-of-session
   report to the user (2–4 sentences): what was committed (verified
   count / numerical count / surveys), which issues were closed,
   what's next, then schedule the next `/loop` iteration per the
   pacing policy in CLAUDE.md (§ "`/loop /solve` pacing"). Then stop
   the turn.

# Anti-patterns (don't do)

- **Skipping the novelty gate.** Reformalizing a result Mathlib
  already has burns the session for nothing. Always loogle +
  leansearch first.
- Writing long planning documents without producing a verified
  artifact.
- Dispatching `librarian` repeatedly without acting on the findings.
- **Dispatching a subagent with no narration line.** The user is
  watching the live turn; every `Agent` dispatch needs a `→` line and
  every return a `←` line (see "Work loop" → narration discipline).
- **On a T1 clone, dispatching a full librarian survey, or opening a
  sub-vector / formalizing a reduction for a "cite it" / "comment on"
  checkbox.** That is the 5.7h overkill failure mode — discharge the
  checkbox at its stated depth (a remark + citation) and move on.
- **Over-deepening a checkbox** (formalizing a result the checkbox only
  asked you to cite/note). Run the scope-fidelity gate first.
- **Leaving sub-issues open at session end.** The ticket sweep
  (Session-end Step 1a) must leave exactly one open issue — the
  ROADMAP. Dangling open vectors were a real failure on a prior clone.
- Restating the difficulty of the target in different words.
- Adding `[sketch]` entries to `proof.tex` faster than they get
  promoted (the ratio should approach 1:1 over many sessions).
- Skipping `critic` review.
- Forgetting to push (every commit must reach GitHub immediately).
- Forgetting to rebuild the PDF after a promotion (and at session
  end) — the early, continuously-fresh PDF is the deliverable that
  earns the longer run.
- Opening a new issue when an existing open one already tracks the
  subgoal — comment on the existing one instead.
- Writing detailed state into STATUS.md when it belongs in an Issue.
- **Promoting a sketch by just flipping the tag.** Promotion is a
  rewrite: replace the sketch block with a publication-style proof,
  pull out auxiliary lemmas, add the theorem footnote, update the
  appendix and abstract. See the "Manuscript style" section. A
  one-character tag flip leaves a scratchpad in the PDF.
- **Leaking project scaffolding into the manuscript** — "Initial
  attack vectors", "Revision history", "Verified results"/"Sketches"
  bins, "Critic review PASS" remarks, inline
  `\texttt{[verified: …]}` text in theorem heads, "Step 1 / Step 2"
  labels in proofs. The manuscript is a paper, not a project log;
  scaffolding lives in the ROADMAP issue, `STATUS.md`, `findings/`,
  and the git log.
- **Accepting `Lean.ofReduceBool` silently.** `native_decide` adds
  this axiom; promotion requires explicit critic sign-off case by
  case.

# When you find a real breakthrough

If you discover a genuinely novel result — not a known theorem
rediscovered, but a step that meaningfully advances the project:

1. Triple-check with `critic` (one round is not enough).
2. Cross-check with `librarian`: has someone already proved this?
   Search precisely; loogle + leansearch + arXiv.
3. Write `findings/breakthrough-<topic>-<date>.md` with the full
   argument.
4. Promote in `manuscript/proof.tex` only after both checks pass.
5. Commit: `BREAKTHROUGH: <one-line>` (all-caps verb signals the
   user should look).
6. Open a new GitHub issue with label `roadmap` only if it shifts
   the strategic posture; otherwise update the existing ROADMAP
   issue prominently.
7. Continue working — a breakthrough usually unlocks a new wave of
   subgoals; act on them in the same session.

# Final notes

Treat each session as a contribution to a long-running collaborative
project. The user is the reviewer; the GitHub log + Issues + PDF are
the conversation. Be concrete, be honest about what's verified vs
sketched, commit often, never pretend.

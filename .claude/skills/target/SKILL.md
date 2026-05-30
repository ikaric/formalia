---
name: target
description: One-time bootstrap for a fresh formalia clone. Asks the user for a target problem, scaffolds the Lake project under `formal/`, seeds `manuscript/proof.tex` with the statement, writes `STATUS.md` and `findings/INDEX.md`, opens the ROADMAP issue (#1) with a seeded sub-goal checklist, and commits. Run this once per cloned repo before `/loop /solve`. This skill IS interactive — it uses AskUserQuestion. Do not call from /solve.
---

You are the **target** skill for a freshly-cloned `formalia`
repository. Your single job is to take a clone that contains the
template harness — but no problem definition — and turn it into a
working project pointed at a specific open mathematical problem.

After this skill completes, the clone is ready for autonomous work
via `/loop /solve`.

# Pre-flight: is this actually a fresh clone?

Run:

```sh
grep -l "FORMALIA_TEMPLATE" manuscript/proof.tex 2>/dev/null
```

If the marker is **absent**, this clone has *already* been set up.
**Refuse to run.** Tell the user that the project has an existing
problem definition and rerunning `/target` would overwrite their
work. Suggest they manually edit `manuscript/proof.tex` if they want
to adjust the problem, or start a fresh clone from the template for
a new problem.

If the marker is **present**, proceed.

Also verify:

```sh
gh auth status 2>&1 | head -3
git remote -v
git config user.name; git config user.email
```

The active `gh` account must be `<GH_USERNAME>` (the value the user
provided to `make init`). The remote must be
`https://github.com/<GH_USERNAME>/<something>.git`. The git user
must be `<GIT_USER_NAME> <<GIT_USER_EMAIL>>`. Fix any of these
before continuing:

- account: `gh auth switch -u <GH_USERNAME> && gh auth setup-git`
- local identity: re-run `make init` (it sets `user.name` and
  `user.email` for the repo), or set them manually with
  `git config user.name "<GIT_USER_NAME>" && git config user.email
  "<GIT_USER_EMAIL>"`.

If `grep -E '<(GH_USERNAME|GIT_USER_NAME|GIT_USER_EMAIL)>' CLAUDE.md`
returns a hit, `make init` was never run on this clone. Refuse to
continue and tell the user to run `make init` first.

# Interactive elicitation

Ask the user in **one** batched `AskUserQuestion` call (the tool caps
at four questions, and we use all four). Phrase the questions tightly
— the user has already chosen to run `/target`, so they know they're
about to define a problem.

1. **Problem display name** (short — used in the ROADMAP issue
   title, the manuscript title, and the Lake module path). Free-text.
   Example: "Frankl's union-closed sets conjecture". The display name
   is converted to a Lake-friendly identifier (CamelCase, no spaces,
   no special chars — e.g., "Frankl" or "FranklUnionClosed").
2. **Problem field** (so the right subagents are flagged as
   relevant). Single-select. Options: analytic number theory / sieve
   methods / additive combinatorics / extremal set theory / discrete
   or combinatorial geometry / other.
3. **Problem statement** (one paragraph). Free-text. The user pastes
   the precise statement. *Guidance text on this field:* "Optionally
   end with up to 3 initial attack vectors, one per line prefixed
   `vector:`, or write nothing to let the librarian propose them."
   (Initial vectors are seeded from these lines; this avoids spending
   a separate question on an optional input.)
4. **Expected scope** (the calibration that prevents overkill).
   Single-select — maps directly to the scope tier (see CLAUDE.md
   §"Scope tiers"):
   - **T1 — known / textbook**: "Formalize exactly the stated claims,
     then halt. One short session (minutes)." Pick this when the asks
     are settled/known and small.
   - **T2 — bounded-hard**: "A finite or known-hard target; enumerable
     sub-lemmas; partial progress counts."
   - **T3 — open problem**: "A genuine open problem; reductions,
     partial results, surveys expected."
   When the user is unsure, **default to T2**. Choose T1 only when the
   asks are explicitly small and settled. The tier is a scope/contract
   choice, never an outcome prediction — do not phrase any option as
   "this can/can't be solved."

If the user has *already* supplied any of this information in the
conversation context above the `/target` invocation, use that
directly and skip the corresponding question(s) — do not bother the
user twice. (If scope is genuinely undeterminable from context, still
ask question 4 — mis-scoping is the failure this calibration exists to
prevent.)

# Compute the Lake identifier

From the display name, derive a CamelCase identifier suitable for a
Lean module path: strip diacritics, drop apostrophes and spaces,
capitalize each word. Examples:

- "Frankl's union-closed sets conjecture" → `Frankl` (short form
  preferred unless ambiguous)
- "Infinitude of primes" → `InfinitudeOfPrimes`
- "Hadwiger–Nelson problem" → `HadwigerNelson`
- "Twin prime conjecture" → `TwinPrime`

If the resulting identifier collides with a Mathlib top-level
namespace, append the problem field as a suffix (e.g., `FranklSets`,
`TwinPrimeNT`).

# Lake project rename

The template ships with `formal/Formalia.lean` and `formal/Formalia/` as
placeholders. Rename them to the per-clone identifier:

```sh
mv formal/Formalia.lean "formal/<Identifier>.lean"
mv formal/Formalia "formal/<Identifier>"
```

Update `formal/lakefile.toml`:
- `name = "Formalia"` → `name = "<Identifier>"`
- `defaultTargets = ["Formalia"]` → `defaultTargets = ["<Identifier>"]`
- `[[lean_lib]]` block's `name = "Formalia"` → `name = "<Identifier>"`

Update `formal/<Identifier>.lean` to be a root module that imports
the per-concept files under `formal/<Identifier>/`:

```lean
-- Root module for the <DisplayName> formalization.
-- Per-concept proofs live under `formal/<Identifier>/`.
import «<Identifier>».Main
```

Create `formal/<Identifier>/Main.lean` as a stub:

```lean
import Mathlib

namespace «<Identifier>»

-- The main theorem will be stated here by /solve once the librarian
-- survey lands and the first sketch is grounded.

end «<Identifier>»
```

(The French-quote `«...»` syntax lets the identifier contain
characters that wouldn't otherwise parse as a Lean name; safe to use
unconditionally.)

# Toolchain verification

**Tell the user up front** (one line, before running anything): "Lean
toolchain + Mathlib cache is a one-time ~3 GB download — the next step
takes several minutes on a fresh machine; this is a download, not a
hang." Then run the steps **separately**, in order, each verified —
do **not** chain `lake exe cache get && lake build` in one Bash call
(a single chained invocation has thrown `Invalid tool parameters`, and
chaining hides which half failed):

**Step A — toolchain (idempotent):**
```sh
elan toolchain install $(cat formal/lean-toolchain)
```

**Step B — pull the prebuilt Mathlib cache, then verify it landed:**
```sh
(cd formal && lake exe cache get)
ls formal/.lake/packages/mathlib/.lake/build/lib >/dev/null 2>&1 \
  && echo "cache OK" || echo "cache MISSING — re-run cache get"
```
If the verify prints `MISSING` or only a partial tree, re-run `cache
get` before building (a partial cache makes Step C recompile Mathlib
from scratch — tens of minutes).

**Step C — build, with the diagnostic:**
```sh
(cd formal && lake build)
```
If this step starts compiling Mathlib modules from scratch (rather
than finishing in seconds against the cached `.olean` files), Step B
did not land — stop and re-run Step B. If `lake build` fails on a
fresh template (the per-concept stub is empty), investigate before
proceeding — a non-buildable template is a bug, not a user problem.

Separately, ensure the Python venv exists:

```sh
uv sync   # creates .venv/ from pyproject.toml + uv.lock
```

If `uv` is not installed: `brew install uv`.

The `.venv/` directory is gitignored — it lives inside the clone but
is never committed. After this step, **all Python invocations** in
this clone go through `.venv/bin/python`, never `python3`.

# Framing rules for seeded content (read before writing anything)

Pre-committed defeatism — phrases like "Realistic outcome
assessment: this vector cannot settle…", a "Hard constraints"
section enumerating what would not be attempted, and per-vector
caps tied to published obstacles — is banned from seeded content.
The user has explicitly rejected this framing. When seeding the
manuscript and ROADMAP body:

- The SOTA / published-obstacle line (parity barrier,
  polynomial-method degree cap, RH-conditionality, 3/8 wall, etc.)
  is a **literature note** with a citation. It is *not* a project
  ceiling. Write it as attribution ("Selberg 1950s identifies the
  parity barrier as the standard obstacle to…") and stop there.
- **Banned framings** (do not emit these into manuscript or issue
  bodies under any circumstances):
  - "Realistic outcome assessment" / "the realistic best deliverable"
    / "the best we can hope for is…"
  - "This vector cannot settle X" / "is not achievable via this lane"
    / "is parity-barrier-blocked" — banned as project statements;
    allowed only as quoted attributions.
  - A "Hard constraints" section. Name the section "Correctness
    rules" and frame each bullet as how surprising results get
    *routed* (critic adversarial review, honest tagging), not as
    what is refused.
- **Required framing** for the attack-vectors section: each vector
  is attempted end-to-end. Published obstacles are noted with a
  citation and a "route through critic if a sub-step appears to
  break this" clause. Honest tagging at promotion time
  (`[verified]`, `[sketch]`, `[conditional]`, `[heuristic]`) is the
  safety net — not editorial pessimism upstream.
- **Keep the seeded manuscript minimal.** Problem statement + empty
  main-result placeholder. **No attack-vector list in the
  manuscript** — that lives in the ROADMAP issue body. No "Realistic
  outcome assessment", no "Hard constraints", no per-vector outcome
  ceilings. Detail accretes in `/solve` sessions through the verify
  cycle, not at `/target`.

These rules are mirrored in `CLAUDE.md`'s Anti-defeatism section; if
the two diverge, `CLAUDE.md` is authoritative.

# File edits

After elicitation:

1. **`manuscript/proof.tex`** — replace the template body. Keep the
   LaTeX preamble (`\documentclass`, package imports, theorem
   environments). Replace the `% FORMALIA_TEMPLATE` marker comment and
   the placeholder body with a **publication-style paper skeleton**
   — title, abstract placeholder, introduction containing the
   problem statement, and a `Main result` section with a placeholder
   marked `% [sketch]` for the first /solve session to fill in:

   ```latex
   \title{<problem display name>}
   \author{<GIT_USER_NAME>}
   \date{\today}

   \begin{document}
   \maketitle

   \begin{abstract}
   \emph{Abstract to be written once the first verified result has
   landed; the introduction below states the problem.}
   \end{abstract}

   \section{Introduction}
   \label{sec:intro}

   <user-supplied paragraph, rewritten into mathematical-exposition
   voice if the supplied phrasing is first-person or project
   commentary. Close with a forward-looking sentence such as: "The
   main result of this manuscript is recorded in
   Section~\ref{sec:main}, currently in preparation.">

   \section{Main result}
   \label{sec:main}

   % [sketch] main result placeholder — to be stated and proved by /solve.
   \emph{Statement and proof in preparation.}

   \end{document}
   ```

   **Banned from the seeded manuscript** (these belong elsewhere):
   - `\section{Initial attack vectors}` / `\section{Roadmap}` / any
     attack-vector list — those go into the ROADMAP issue body
     (parking lot) and into per-vector issues, not the manuscript.
   - `\section{Revision history}` — the git log is the history.
   - `\section{Status}` / `\section{Current focus}` — `STATUS.md`
     carries the live pointer.
   - `\section{Verified results}` / `\section{Sketches}` —
     pre-emptive container sections. Theorems live where the
     mathematical exposition puts them, with a `% [sketch]` or
     `% [verified: ...]` comment above each, not in segregated bins.
   - Inline `[verified: formal/X.lean]` text in section or theorem
     headings — verification status is tracked by a LaTeX comment
     above the theorem (machine-readable, invisible in PDF) plus an
     optional footnote on the statement, per the manuscript-style
     policy in `.claude/skills/solve/SKILL.md`.

   The manuscript reads like an arXiv preprint from the first
   commit; project scaffolding lives in the ROADMAP issue,
   `STATUS.md`, `findings/`, and the git log.

2. **`STATUS.md`** — overwrite with the initial-state pointer:

   ```
   # Status

   Current focus: project just initialized; first /solve session
   will run librarian survey + grounded sketch.

   Last commit: target commit (this commit).

   Blocker: none.

   Problem: <problem display name>.
   ```

3. **`README.md`** — overwrite the template README with a
   **problem-specific entry page**. The template README documents
   the formalia harness; once the clone is targeted at a concrete
   problem, the README should instead serve as the landing page
   for anyone (the user, a collaborator, a passing reader) who
   opens the repo on GitHub.

   Use this scaffold verbatim, substituting the problem-specific
   values you elicited:

   ````markdown
   # <problem display name>

   > <one-sentence statement of the problem>.

   Autonomous Lean 4 proof attempt against
   [Mathlib](https://github.com/leanprover-community/mathlib4),
   driven by the [formalia](https://github.com/<GH_USERNAME>/formalia)
   harness. The Lean kernel is the trust root: every claim tagged
   `[verified]` compiles under `lake build` with `#print axioms`
   reporting only the three foundational axioms.

   ## Problem

   <one or two paragraphs restating the problem in mathematical
   exposition voice — matches the introduction of
   `manuscript/proof.tex`. GitHub renders inline `$...$` and
   display `$$...$$` LaTeX.>

   ## Status

   <!-- BEGIN STATUS -->
   - **Last session**: project just initialised (`<YYYY-MM-DD>`)
   - **Roadmap**: [#1](../../issues/1) — sub-lemma checklist with progress meter
   - **Verified results**: 0 / TBD (first `/solve` session will populate)
   - **Current focus**: librarian survey + grounded initial sketch
   - **Manuscript**: [`proof.pdf`](manuscript/proof.pdf) — rebuilt per promotion
   <!-- END STATUS -->

   ## What's verified

   <!-- BEGIN VERIFIED -->
   *No verified results yet — first `/solve` session has not run.*
   <!-- END VERIFIED -->

   ## What's open

   <!-- BEGIN OPEN -->
   See the [ROADMAP issue](../../issues/1) for the live sub-lemma
   checklist. Sub-issues carry labels (`open`, `sketch`,
   `numerical`, `conditional`, `survey`, `review`, `deadend`)
   indicating their verification state.
   <!-- END OPEN -->

   ## Assessment

   <!-- BEGIN ASSESSMENT -->
   - **Target progress**: 0 / TBD sub-lemmas verified
   - **Verified vs sketched**: 0 verified, 0 sketched, 0 numerical
   - **Obstructions logged**: none yet
   - **What the evidence shows**: project just initialised; no attempts made.
   <!-- END ASSESSMENT -->

   ## Timeline

   <!-- BEGIN TIMELINE -->
   *No sessions yet — the first `/solve` session will append the first entry.*
   <!-- END TIMELINE -->

   ## Repository tour

   - [`manuscript/proof.tex`](manuscript/proof.tex) /
     [`proof.pdf`](manuscript/proof.pdf) — the canonical evolving paper
   - [`formal/<DisplayName>/`](formal/<DisplayName>/) — Lean 4
     theorems (`#print axioms` checked, no `sorry` outside
     `[sketch]`-tagged files)
   - [`findings/`](findings/) — inter-agent notes, literature
     surveys, reviewer reports, dead ends, strategic pivots
   - [`STATUS.md`](STATUS.md) — brief pointer to the last session
     and current focus
   - [ROADMAP](../../issues/1) — live work-queue dashboard with
     checkbox progress meter

   ## Harness

   This clone runs the
   [formalia](https://github.com/<GH_USERNAME>/formalia)
   autonomous-proof harness: twelve specialist subagents under
   `.claude/agents/`, three slash-commands (`/target`, `/solve`,
   `/vector`) under `.claude/skills/`, a mandatory novelty gate
   against Mathlib, and honest tagging
   (`[verified]` / `[sketch]` / `[conditional on H]` /
   `[numerical: range]` / `[heuristic]`).

   The Status / Verified / Open blocks above are bounded by HTML
   comment sentinels and are refreshed automatically at the end of
   every `/solve` session — do not hand-edit between the markers.
   ````

   **Required invariants of the per-clone README:**
   - The **five** sentinel pairs `<!-- BEGIN STATUS -->` /
     `<!-- END STATUS -->`, `<!-- BEGIN VERIFIED -->` /
     `<!-- END VERIFIED -->`, `<!-- BEGIN OPEN -->` /
     `<!-- END OPEN -->`, `<!-- BEGIN ASSESSMENT -->` /
     `<!-- END ASSESSMENT -->`, and `<!-- BEGIN TIMELINE -->` /
     `<!-- END TIMELINE -->` must remain in the file (exact spelling,
     one per line) — `/solve`'s session-end protocol locates them
     by string match and rewrites the content between them.
   - The `<DisplayName>` in path links must be the per-clone Lake
     identifier you computed in the "Compute the Lake identifier"
     step (not the literal string `<DisplayName>`).
   - The harness link target `<GH_USERNAME>/formalia` should be
     the value already substituted by `make init`. If the bare
     placeholder still appears, `make init` was never run — abort
     and tell the user.

4. **`findings/INDEX.md`** — create it with a header line and the
   conventions list:

   ```
   # Findings index

   Inter-agent notes drop here. Naming conventions:

   - `lit-<topic>-<date>.md` — literature survey (librarian)
   - `heur-<topic>-<date>.md` — heuristic / numerical
     (computationalist)
   - `deadend-<topic>-<date>.md` — abandoned approach
   - `openq-<topic>-<date>.md` — open question surfaced during work
   - `review-<topic>-<date>.md` — critic adversarial review
   - `decision-<topic>-<date>.md` — orchestrator decision recorded
     without user input
   - `pivot-<date>.md` — strategic pivot ceremony (`/vector pivot`)

   (empty — first /solve session will populate)
   ```

5. **(Re-build the PDF.)** `(cd manuscript && tectonic -X compile
   proof.tex)` so `manuscript/proof.pdf` exists from the start.
   Commit it.

# GitHub: create labels

Before opening any issues, ensure the project's labels exist. A
fresh template clone has zero labels — `gh issue create --label
<name>` errors out if a referenced label is missing. The loop below
is idempotent (each `gh label create` returns non-zero if the label
already exists; the trailing `|| true` keeps the loop alive on
re-runs).

```sh
for spec in \
  "roadmap:8B5CF6:Top-level tracker (issue #1)" \
  "verified:0E8A16:Lean compiled, #print axioms clean, critic PASS" \
  "numerical:1D76DB:Computational verification on a finite range" \
  "sketch:FBCA04:Informal argument; ready for formalization" \
  "heuristic:F9D0C4:Empirical pattern only" \
  "conditional:5319E7:Depends on a named open hypothesis" \
  "survey:0075CA:Literature survey deliverable" \
  "review:D93F0B:Critic adversarial review" \
  "open:CFD3D7:Subgoal not yet started" \
  "deadend:6B7280:Approach abandoned with documented reason" \
  "deprioritized:D4D4D4:Retired vector / approach (deferred, not dead)" \
  "vector:FBCA04:Live attack vector (managed by /vector)" \
  "formal-lean:1A237E:Lean formalization work" \
  "formal-numerics:3F51B5:Python / computational work" \
  "novel:B60205:Verified result that is genuinely new (critic+librarian confirmed)" \
  "formalization:5319E7:Verified port of a known result not yet in Mathlib" \
  "known-cited:C5DEF5:Closed by importing/citing a result already in Mathlib/Reservoir" \
  "tier-T1:0E8A16:Scope tier T1 — known/textbook" \
  "tier-T2:FBCA04:Scope tier T2 — bounded-hard" \
  "tier-T3:D93F0B:Scope tier T3 — open problem"
do
  IFS=':' read -r name color desc <<< "$spec"
  gh label create "$name" --color "$color" --description "$desc" 2>/dev/null || true
done

# If the user supplied initial attack vectors (up to 3), also create
# the per-vector labels so the per-vector issues below can use them.
# /vector creates additional vector-V<N> labels on demand.
for n in 1 2 3; do
  gh label create "vector-V$n" --color FBCA04 --description "Attack vector V$n" 2>/dev/null || true
done
```

The full label set is documented in CLAUDE.md's progress-tracking
section. Creating them all up-front (rather than only the ones used
right now) means `/solve` and `/vector` won't hit this gap when they
start applying these labels in subsequent sessions.

# GitHub: open ROADMAP issue

Open issue #1 — the ROADMAP — using `gh issue create`. Add the
`roadmap` label **and** the `tier-T<k>` label for the elicited scope
tier.

**First, decompose the user's statement into its literal asks** — one
checkbox per thing the user actually asked for, `N` = the number of
literal asks. A "comment on" / "note that" / "as bonus commentary" ask
becomes **one** checkbox whose deliverable is a short manuscript remark
plus a citation — **never** a formalization vector and **never**
expanded into sub-lemmas. This 1:1 decomposition is the ROADMAP
contract. (Worked example for "∑ pₙ/n! converges; note it has no closed
form; relate to e": exactly 3 checkboxes — `converges` = 1 Lean
theorem; `no closed form` = 1 cited remark; `relation to e` = 1
paragraph. N=3, halts at 3/3. The full irrationality reduction is out
of scope — that is the overkill this decomposition prevents.)

Title: `ROADMAP — <problem display name>`

Body uses **GitHub task-list syntax** so checkboxes auto-tick when
linked issues close (`- [ ] #N <title>`). Branch the body and the
seeded issues on the scope tier:

**T1 (known/textbook):** Do **not** seed a standing librarian-survey
issue and do **not** seed an `## Attack vectors` section — the survey
issue is the exact entry point that escalates a trivial problem into
hours of work. Open **one issue per literal ask** (small — usually
≤3), labelled `open` + the right kind label (`formal-lean` for a claim
to prove, otherwise plain `open` for a remark/citation ask). The first
`/solve` session rips through them (novelty gate → formalize or cite →
promote → rebuild PDF) and halts at `N/N` in a single short session.
Body:

```markdown
## Scope tier: T1 — known/textbook (formalize the stated asks, then halt)

## Problem

<user-supplied paragraph>

## Top-3 next subgoals (refreshed each session end)

1. [#2 <first literal ask>](../issues/2) — <one-line deliverable>
2. [#3 <second literal ask>](../issues/3) — <one-line deliverable>
3. [#4 <third literal ask>](../issues/4) — <one-line deliverable>

## Required sub-lemmas

- [ ] #2 <first literal ask> — <deliverable: e.g. "1 Lean theorem">
- [ ] #3 <second literal ask> — <deliverable: e.g. "cited remark">
- [ ] #4 <third literal ask> — <deliverable: e.g. "one paragraph">

## Progress

*0 / N sub-lemmas closed.*
```

**T2 / T3 (bounded-hard / open):** Seed the literal-ask checkboxes
**and** the librarian survey (issue #2) **and**, if the user supplied
vectors, the per-vector issues. Body:

```markdown
## Scope tier: T<2|3> — <bounded-hard | open problem>

## Problem

<user-supplied paragraph>

## Top-3 next subgoals (refreshed each session end)

1. [#2 librarian survey](../issues/2) — first /solve dispatches the librarian
2. (blank — populated next session)
3. (blank — populated next session)

## Required sub-lemmas

- [ ] #2 Literature survey: SOTA + 3 closest predecessors + 3 attack vectors
- [ ] (one checkbox per literal ask; more added by /solve as work progresses)

## Attack vectors

- [ ] #3 V1: <user vector 1 — if supplied>
- [ ] #4 V2: <user vector 2 — if supplied>
- [ ] #5 V3: <user vector 3 — if supplied>
- (or, if user said "let librarian propose": "_to be proposed by
  librarian in first /solve session_")

## Retired vectors

(none yet)

## Progress

*0 / N sub-lemmas closed.*
```

**For T2/T3 only**, then open issue #2 — the librarian survey — with
labels `open + survey`:

Title: `Literature survey — <problem display name>: SOTA bounds and attack vectors`

Body:

```markdown
First /solve session: dispatch the librarian agent to:
1. Run the **novelty gate**: loogle + leansearch to check whether
   Mathlib already has the result (or a close cousin).
2. Locate the closest published bound on the target problem.
3. Identify the last 3 years of arXiv activity on the topic.
4. Propose 3 candidate attack vectors with one-line rationale each.
5. Seed `findings/research-state.md` with the known-vs-open picture
   and a per-checkbox novelty hint.

Output: `findings/lit-<topic>-<date>.md` + `findings/research-state.md`
referenced from this issue.
```

For T2/T3, if the user supplied concrete attack vectors at elicitation
time (the `vector:`-prefixed lines on the problem-statement field),
also open issues #3..#(2+k) — one per vector — with labels `open +
vector + vector-V<N>` (auto-incrementing N starting at 1):

Title: `V<N>: <vector name>`

Body:

```markdown
Attack vector V<N> seeded by /target.

<user-supplied vector statement>.

First /solve session: librarian to ground this in literature
(`findings/lit-V<N>-<date>.md`); then domain agent to produce a
sketch; then formalist + critic to attempt the verify cycle.
```

# Commit & push

Commit (single commit is fine — bootstrap is one work unit):

```
target: initialize for <problem display name>

Bootstrap of a fresh formalia clone.
- Lake project renamed Formalia → <Identifier>.
- Problem statement seeded in manuscript/proof.tex.
- Scope tier: T<k>.
- GitHub labels created (roadmap, verified, sketch, survey, novelty, tiers, …).
- ROADMAP opened as issue #1 (tier-T<k>) with the literal-ask checklist.
- (T2/T3) Librarian-survey scheduled as issue #2; <k> vector issues opened.
- (T1) One issue per literal ask opened.
Ready for /loop /solve.
```

Push to `origin/main`.

# T1 fast-completion (one-shot for already-formalized asks)

**T1 clones only.** After the bootstrap commit, run the **novelty
gate** — loogle + leansearch via `WebFetch`, the cheap direct queries,
*not* the heavyweight `librarian` agent — on each literal ask. The
goal: if the asks already live in Mathlib, `/target` should finish the
whole thing here, so a textbook target (e.g. "there are infinitely
many primes" → `Nat.exists_infinite_primes`) becomes a one-shot:
bootstrap → import wrapper → built PDF → closed ROADMAP, no separate
`/solve` needed.

For each literal ask the gate confirms is **already in Mathlib** (a
pure import/re-export, no real proof obligation):

1. Write `formal/<Identifier>/<Ask>.lean` importing the Mathlib module
   and re-exporting the result under `namespace «<Identifier>»`. Build:
   `(cd formal && lake build)`. Capture `#print axioms <name>` — it
   must list only the three foundational axioms.
2. Promote the manuscript entry to
   `% [verified: formal/<Identifier>/<Ask>.lean] [known-cited]` with
   the Mathlib module path cited (follow the promotion-rewrite policy
   in `.claude/skills/solve/SKILL.md` § Manuscript style — this is a
   real rewrite, not a tag flip), and add the theorem footnote.
3. Rebuild the PDF; commit `formalize:`, `promote:`, `pdf:` (separate
   commits, push each). Close the ask's issue (`Closes #N`), labels
   `known-cited` + `verified`, drop `open`.

A pure import wrapper of a Mathlib result does not need a full `critic`
round — confirm by hand that the imported lemma states exactly the
claim and that `#print axioms` is clean. **Anything beyond a pure
re-export — a result that needs an actual (even short) proof — is NOT
done here.** Leave its issue `open` and hand it to the first `/solve`
session (its T1 fast-path finishes it with the full verify cycle,
critic included). `/target` does import wrappers, not proofs.

**If every literal ask closed this way** (a fully-in-Mathlib T1
target), the clone is already at `N/N`: refresh the README STATUS /
VERIFIED / ASSESSMENT / TIMELINE blocks and the ROADMAP `## Progress`
line, run the ticket sweep (everything closed except the ROADMAP),
commit `status: target reached at /target — N/N (all asks already in
Mathlib)`, push, and report **target reached** (see below). No `/solve`
invocation is required.

# Report back to user

**If the T1 one-shot completed the target** (`N/N`, all asks closed):
output 3–4 sentences — problem display name + ROADMAP URL; "target
reached at bootstrap — all asks were already in Mathlib (cite the
module path[s])"; built `proof.pdf` is committed; "nothing to run —
inspect the PDF, or `/vector add` to extend the scope." Do **not**
tell the user to run `/loop /solve` in this case.

**Otherwise** (T2/T3, or a T1 with asks still needing proof): output
3–4 sentences — problem display name + ROADMAP URL; number of issues
seeded (and, for T1, how many asks the novelty gate already closed as
import wrappers); "Run `/loop /solve` to begin work."

Then stop. **Do not start the open-ended work loop yourself** — the
T1 import-wrapper one-shot above is the only work `/target` performs;
all sketch/proof/critic work belongs to `/solve`.

# Anti-patterns

- Don't write any manuscript content beyond the problem statement
  (placed in the Introduction) and the empty `Main result`
  placeholder. No attack-vector list, no revision history, no
  "Verified results"/"Sketches" bins — those are project-log
  structures, banned from the manuscript. The attack-vector parking
  lot lives in the ROADMAP issue body. The first /solve session
  does the literature survey and produces the first real content.
- Don't emit a "Realistic outcome assessment", a "Hard constraints"
  section, or per-vector ceilings ("cannot settle", "blocked by",
  "not achievable via this lane"). See the **Framing rules for
  seeded content** section above. The user has previously rejected
  this framing in writing; reproducing it is a `/target` failure.
- Don't open more than 5 issues at `/target` time. ROADMAP +
  librarian survey + up to 3 vector issues is enough; more arrives
  organically as /solve runs.
- Don't pre-decide which subagent will work first. /solve picks
  based on the open issue queue.
- Don't dispatch the `librarian` agent yourself. `/target` is
  bootstrap; survey work belongs to a /solve session. (The **cheap
  novelty gate** — loogle + leansearch via `WebFetch` — is the one
  exception, run for T1 clones in the fast-completion step above; that
  is not the librarian agent.)
- Don't write to `formal/<Identifier>/` beyond `Main.lean`'s empty
  stub — **except** the T1 fast-completion import wrappers above, which
  are pure Mathlib re-exports. Any result needing an actual proof is
  left for the first /solve session.
- Don't write to `findings/` beyond the INDEX.md.
- Don't pre-install Mathlib-adjacent packages (Reservoir packages,
  SAT solvers, sage) at `/target` time. The first subgoal that
  needs them installs them on demand.

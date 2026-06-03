<a id="top"></a>
<div align="center">

# Formalia

### Autonomous Lean 4 proof harness for open mathematical problems

[Quick Start](#-quick-start) • [How It Works](#-how-it-works) • [Agents](#-the-agents) • [Skills](#-the-skills) • [Verification](#-verification-gates) • [FAQ](#-faq)

[![Lean 4](https://img.shields.io/badge/Lean-4.30.0-1A237E?logo=lean&logoColor=white)](https://leanprover.github.io)
[![Mathlib](https://img.shields.io/badge/Mathlib-pinned%20per%20clone-3F51B5)](https://github.com/leanprover-community/mathlib4)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-agent%20harness-D97757)](https://docs.claude.com/claude-code)
[![GitHub Template](https://img.shields.io/badge/GitHub-Template-2EA44F?logo=github&logoColor=white)](https://github.com/<GH_USERNAME>/formalia/generate)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

</div>

---

## Overview

Formalia is a GitHub template for attacking open mathematical problems with an LLM-driven proof harness whose trust root is the **Lean 4 kernel**. Clone the template, point it at a problem, and run the autonomous loop.

- **Lean kernel is the trust root** — every `[verified]` claim is checked by `lake build` plus `#print axioms` against the three Lean foundational axioms
- **Mathlib reuse before reproof** — a mandatory novelty gate (loogle + leansearch) makes every subgoal start with "is this already formalized?"
- **Twelve specialist agents** — analyst, sieve theorist, combinatorialist, geometer, algebraist, topologist, probabilist, complexity theorist, formalist, computationalist, librarian, critic — dispatched in parallel when independent, and fanned out into **agent fleets** (adversarial critic panels, parallel research sweeps) via the Workflow orchestrator
- **Four slash-commands** — `/target` (bootstrap), `/solve` (autonomous loop), `/vector` (vector lifecycle), `/polish` (final manuscript pass)
- **Honest tagging discipline** — every claim is `[verified]` / `[sketch]` / `[conditional]` / `[numerical]` / `[heuristic]`, and every verified result is further classified `novel` / `formalization` / `known-cited` so a reader can tell a genuine contribution from a re-proof of a known fact
- **Scope calibration** — `/target` sets a scope tier (T1 known/textbook · T2 bounded-hard · T3 open) so a textbook problem finishes in minutes and is never over-formalized past what was asked
- **Publication-quality manuscript** — `manuscript/proof.tex` reads like an arXiv preprint, with `proof.pdf` rebuilt by `tectonic` after every promotion (not just at session end), so the deliverable is readable early and keeps growing
- **GitHub-Issues work queue** — ROADMAP issue with checkbox-tracked sub-lemmas; at rest exactly one issue is open (the ROADMAP); one repo per problem

The deliverable is a machine-checked Lean 4 proof and a publication-style PDF. LLM output is intuition, not proof; the kernel decides.

---

## 🚀 Quick Start

**Prerequisites:** [`elan`](https://github.com/leanprover/elan) (Lean toolchain manager), [`uv`](https://docs.astral.sh/uv/) (Python), [`tectonic`](https://tectonic-typesetting.github.io/) (LaTeX → PDF), [`gh`](https://cli.github.com/) (GitHub CLI), [Claude Code](https://docs.claude.com/claude-code)

```bash
# 1. Clone the template into a new private repo on your account.
#    Replace <upstream-owner> with whoever hosts the template you're
#    cloning (typically the upstream you forked from, or your own
#    account if you keep your own fork of the template).
gh repo create --template <upstream-owner>/formalia <your-username>/<problem-name> --private --clone
cd <problem-name>

# 2. One-time per-clone setup. Prompts for your GitHub username, full
#    name, and email; substitutes the GH_USERNAME, GIT_USER_NAME, and
#    GIT_USER_EMAIL placeholders across the template; writes the
#    repo-local git config. Idempotent — safe to re-run.
#
#    Non-interactive form:
#      GH_USERNAME=foo GIT_USER_NAME='Foo Bar' \
#        GIT_USER_EMAIL=foo@bar.example make init
make init

# 3. Bootstrap — defines the problem, scaffolds the Lake project, opens
#    ROADMAP, and replaces this README with a problem-specific entry
#    page (status / verified / open blocks refreshed each session).
#    Interactive; uses AskUserQuestion. (In Claude Code.)
/target

# 4. Run autonomously. Designed for unattended sessions; the harness
#    commits and pushes per work unit, narrates each step in the live
#    turn, and rebuilds proof.pdf on every promotion (readable early,
#    not after hours). Halts on its own when the ROADMAP shows N/N
#    closed OR the seeded lines are exhausted, sweeping all sub-issues
#    closed so only the ROADMAP stays open (see "anti-overreach" in
#    CLAUDE.md). A trivial T1 target may already be complete after
#    /target alone — in that case there's nothing to run here.
/loop /solve

# 5. (Optional, after the loop halts.) Polish the manuscript for
#    publication — rewrite the abstract, expand the introduction,
#    attach footnote-links from each theorem to its Lean file,
#    add the "Formal verification" appendix with verbatim Lean
#    snippets, rebuild the PDF. Mostly autonomous; one commit.
/polish
```

Between sessions, when the strategic landscape shifts, run `/vector` to add, retire, or pivot attack vectors.

> First `/solve` session fetches Mathlib's prebuilt `.olean` cache (~3 GB, ~5 min on first machine). Subsequent builds are incremental.

---

## 🔧 How It Works

The harness encodes one named pattern: the **verify cycle**. Each iteration produces either a verified Lean theorem or an honestly-tagged dead-end.

```mermaid
flowchart LR
    A[Novelty gate<br/>loogle + leansearch] --> B{In Mathlib?}
    B -->|yes| C[Thin import wrapper]
    B -->|no| D[Librarian survey]
    D --> E[Domain agent sketch]
    E --> F[Formalist .lean file]
    F --> G[lake build<br/>+ #print axioms]
    G --> H[Critic review]
    H -->|PASS| I[Promote to verified<br/>3 commits + push]
    H -->|FAIL| J[Downgrade tag<br/>findings/review-*.md]
    C --> I

    style A fill:#1A237E,color:#fff
    style G fill:#1A237E,color:#fff
    style I fill:#2EA44F,color:#fff
    style J fill:#D32F2F,color:#fff
```

The **novelty gate** (step 0) is non-optional. Mathlib formalizes a vast amount of mathematics; before formalizing *anything*, the harness queries `loogle.lean-lang.org` (type-shaped queries) and `leansearch.net` (English queries). If Mathlib has the result, the subgoal collapses to a one-line import wrapper — one ROADMAP checkbox ticked in one commit, no reformalization. Skipping the gate is the documented anti-pattern: sessions get burned reformalizing folklore that already lives upstream.

### Honest tagging

Every claim in `manuscript/proof.tex` carries a single-line LaTeX comment recording its verification status:

| Tag | Meaning |
|---|---|
| `% [verified: formal/X.lean]` | `lake build` succeeds, `#print axioms` lists only the three Lean foundational axioms, critic PASS |
| `% [sketch]` | Informal argument present (or being written). Not yet formalized |
| `% [conditional on H]` | Verified *conditional* on a named open hypothesis (RH, GRH, etc.) |
| `% [numerical: <range>]` | Verified for a finite range by a script under `formal/numerics/` |
| `% [heuristic]` | Empirical pattern only |

The comment is invisible in the rendered PDF but `grep`-able for future sessions. A discreet footnote on the main theorem points to the `.lean` file with the Mathlib commit SHA baked in for reproducibility.

---

## 🧠 The Agents

Twelve specialists live under `.claude/agents/`. The orchestrator (`/solve`) dispatches them by subgoal type; they run in parallel when independent.

```mermaid
flowchart TB
    subgraph orchestrator[Orchestrator]
        Solve["/solve"]
    end

    subgraph crosscutting[Cross-cutting]
        Lib[librarian]
        Form[formalist]
        Comp[computationalist]
        Crit[critic]
    end

    subgraph domain[Domain specialists]
        Ana[analyst]
        Sie[sieve-theorist]
        Cmb[combinatorialist]
        Geo[geometer]
        Alg[algebraist]
        Top[topologist]
        Pro[probabilist]
        Cpx[complexity-theorist]
    end

    Solve --> Lib
    Solve --> domain
    Solve --> Form
    Solve --> Comp
    Solve --> Crit

    style orchestrator fill:none,color:#fff
    style crosscutting fill:none,color:#fff
    style domain fill:none,color:#fff
```

### Cross-cutting agents

| Agent | Specialty | When dispatched |
|---|---|---|
| `formalist` | Lean 4 + Mathlib formalization, tactic guidance, axiom checks | Every promotion candidate |
| `computationalist` | Python (`uv`) scripts, `gmpy2`, SAT runners | Finite-range computational evidence |
| `librarian` | arXiv + Mathlib + Reservoir + MathOverflow + Polymath | Before any non-trivial sketch |
| `critic` | Adversarial review, axiom whitelist, famous-hypothesis check | After every formalization, before promotion |

### Domain specialists

| Agent | Specialty | When dispatched |
|---|---|---|
| `analyst` | Analytic NT: circle method, exp sums, L-functions, zero-free regions | NT subgoal with asymptotic content |
| `sieve-theorist` | Brun / Selberg / large sieve / GPY-Maynard | Counting primes / almost-primes in AP or short intervals |
| `combinatorialist` | Sumsets, Plünnecke–Ruzsa, Freiman, Green–Tao transference | Additive combinatorics, density bounds, AP-counting |
| `geometer` | Unit distance, chromatic numbers, incidence bounds, packings | Discrete / combinatorial geometry |
| `algebraist` | Groups, rings, fields, representations, Galois theory | Algebraic structure / invariant inequalities |
| `topologist` | Homology, knot invariants, Borsuk–Ulam, simplicial methods | Topological invariants / topological combinatorics |
| `probabilist` | Chernoff / Azuma / Talagrand, LLL, entropy method | Concentration claims, probabilistic-method existence |
| `complexity-theorist` | SAT / SMT encodings, LRAT certificates, reductions | Finite search via SAT, Lean LRAT bridging |

> **Per-clone extensibility**: a clone that needs a missing specialist (e.g., `logician`, `category-theorist`, `algebraic-geometer`, `differential-geometer`) drops a new `<name>.md` into `.claude/agents/`. The orchestrator picks it up automatically.

### Orchestration & parallelism

`/solve` is the **spine**: it picks the subgoal, decides which specialists to wake, collects their results, runs the verify cycle, and owns every durable side-effect (commits, `gh issue`, the manuscript, the PDF). The agents never touch Issues or git themselves — they leave notes in `findings/` and report back. Independent agents run **concurrently**: a one-message batch of several `Agent` calls, each in its own context window, results gathered when they return.

For two recurring fan-out shapes the harness escalates from "a few parallel agents" to a deterministic **agent fleet**, driven by a Workflow script that spawns the fleet, validates each return against a schema, and reuses the existing agent definitions verbatim:

| Fleet | Shape | When |
|---|---|---|
| **Adversarial critic panel** | One `critic` per *lens* (hidden open-hypothesis dependence, stronger-than-target, WLOG/"clearly" expansion, sign/scale, parity barrier, …); **any** credible refutation blocks the promotion | A high-stakes promotion — the main theorem, a `novel` claim, or a result that brushes a published wall |
| **Parallel research sweep** (research fleet) | Several `librarian`-class agents each search a *different angle* (Mathlib, arXiv, MathOverflow, Polymath, recent SOTA) at once, then one agent synthesizes the cross-checked findings | A broad literature survey that would be slow as one serial pass |

Five distinct lenses (or angles) catch what five identical passes would miss; the orchestrator still re-runs the authoritative `lake build` + `#print axioms` check itself rather than trusting any agent's self-report.

> **One build at a time.** Agents run in parallel, but **Lean builds are serialized**: Lake has no inter-process build lock (the `lake.lock` was removed before Lean 4.0.0 ever shipped), so two `lake build`s in the shared `formal/.lake/` would race and corrupt each other's `.olean` files rather than wait politely. A *single* `lake build` is already multi-core, so this costs nothing — the harness simply never dispatches two build-running agents at once. Parallel agents that only *write* `.lean` files, run read-only `lake env lean` checks, search the literature, or run numerics proceed concurrently; the actual `lake build` is funnelled through one builder. (Details in `.claude/skills/solve/SKILL.md` § Parallelism.)

---

## 🎛 The Skills

Four slash-commands live under `.claude/skills/`. Two interactive, two autonomous.

| Skill | Mode | Purpose |
|---|---|---|
| **`/target`** | Interactive (`AskUserQuestion`) | One-time bootstrap. Asks for problem name + field + statement + **scope tier** (T1/T2/T3). Renames the Lake project from the `Formalia` placeholder. Seeds the manuscript, opens ROADMAP + (for T2/T3) librarian-survey issues, with the ROADMAP decomposed 1:1 to the literal asks. For a T1 target whose asks are already in Mathlib, runs the novelty gate and completes the one-shot (import wrapper → built PDF → closed ROADMAP) — no `/solve` needed. **Run once per clone.** |
| **`/solve`** | Autonomous (no user input) | Resumes from ROADMAP + open Issues + git log. Picks the next subgoal, runs the novelty gate, dispatches sub-agents (narrating each in the live turn), runs the verify cycle, commits + pushes, rebuilds `proof.pdf` per promotion. Wrap with `/loop /solve` for sustained sessions. Halts on `N/N closed` **or** exhaustion of the seeded lines, sweeping every sub-issue closed so only the ROADMAP remains open — see CLAUDE.md § "Anti-overreach". |
| **`/vector`** | Interactive (`AskUserQuestion`) | Three modes — `add` (seed a new vector), `retire` (close with reason → `deadend` or `deprioritized`), `pivot` (multi-retire + strategic rationale + replacements, with a `findings/pivot-DATE.md` ceremony). |
| **`/polish`** | Autonomous (no user input) | Final-pass manuscript polish, run after the loop halts. Audits the `.tex` against publication standards (abstract written, introduction structured, every theorem has a footnote linking to its Lean file, no scaffolding leftover), adds or refines the "Formal verification" appendix with verbatim Lean snippets, runs `critic` over the polished draft, rebuilds the PDF. One commit. |

### Typical session flow

```mermaid
sequenceDiagram
    participant U as User
    participant T as /target
    participant L as /loop /solve
    participant V as /vector

    U->>T: clone + /target
    T->>T: rename Lake project<br/>seed manuscript<br/>open ROADMAP
    T-->>U: ROADMAP URL

    U->>L: /loop /solve (unattended)
    loop verify cycle
        L->>L: novelty gate
        L->>L: librarian / domain agents
        L->>L: formalist → lake build
        L->>L: critic review
        L->>L: promote + commit + push
    end
    L-->>U: rendered proof.pdf<br/>+ closed issues

    U->>V: /vector pivot (between sessions)
    V-->>U: retired old + seeded new vectors
```

---

## ✅ Verification Gates

What makes a theorem `[verified]`:

1. **`lake build` exits 0** — the `.lean` file compiles against the pinned Mathlib SHA.
2. **`#print axioms <ThmName>`** lists **only**:
   - `propext` (propositional extensionality)
   - `Classical.choice` (classical choice / excluded middle)
   - `Quot.sound` (quotient soundness)

   These three are how Mathlib certifies its theorems. Their presence is normal.
3. **`critic` PASS** — adversarial review (stronger-than-target test, famous-hypothesis check, WLOG/clearly expansion, uniformity check, parity-barrier check, sign / scale check).

### Disqualifying axioms

| Axiom | Why disqualifying |
|---|---|
| `sorryAx` | A `sorry` is in the dependency chain. The theorem is asserted, not proved. Send back to formalist. |
| `Lean.ofReduceBool` | A `native_decide` is in the chain. The proof depends on compiled-Lean evaluation, not on the kernel's reduction. Allowed **case-by-case** after explicit critic review, not by default. |
| Any project-local `axiom <name> : <type>` | Hard no — disqualifies promotion. |

Any of these in the `#print axioms` output keeps the claim at `[sketch]` regardless of what else is true.

---

## 📁 Project Structure

```
your-clone/
├── CLAUDE.md                          # Project operating instructions (loaded every session)
├── STATUS.md                          # Brief live pointer (last session, focus, blocker)
├── README.md                          # User-facing docs
├── LICENSE                            # MIT
├── Makefile                           # `make init` / `make help`
├── pyproject.toml + uv.lock           # Python deps (managed by uv)
│
├── scripts/
│   └── init.py                        # One-time per-clone setup (run via `make init`)
│
├── .claude/
│   ├── agents/                        # 12 subagent definitions
│   └── skills/
│       ├── target/SKILL.md            # One-time bootstrap
│       ├── solve/SKILL.md             # Autonomous-session skill
│       ├── vector/SKILL.md            # Interactive vector add/retire/pivot
│       └── polish/SKILL.md            # Final-pass manuscript polish
│
├── manuscript/
│   ├── proof.tex                      # Canonical evolving manuscript
│   └── proof.pdf                      # Rendered PDF (committed; rebuilt per promotion + session end)
│
├── formal/                            # Lake project root
│   ├── lakefile.toml                  # Mathlib pin + library config
│   ├── lean-toolchain                 # Lean version pin
│   ├── lake-manifest.json             # Lockfile (managed by lake)
│   ├── <DisplayName>.lean             # Root module (renamed by /target)
│   ├── <DisplayName>/                 # Per-concept .lean files
│   └── numerics/                      # Python scripts (computationalist)
│
└── findings/                          # Agent-to-agent markdown notes
    └── INDEX.md
```

---

## 🛠 Setup Requirements

| Tool | Why | Install |
|---|---|---|
| **Lean 4 + Mathlib** (via `elan`) | Formal proofs | `curl https://raw.githubusercontent.com/leanprover/elan/master/elan-init.sh -sSf \| sh -s -- -y` — project pins the toolchain via `formal/lean-toolchain` |
| **Python 3** (via `uv`) | Numerical scripts | `brew install uv`; `/target` runs `uv sync` to create `.venv/` |
| **`tectonic`** | LaTeX → PDF (manuscript rebuild) | `brew install tectonic` |
| **`gh` CLI** | GitHub Issues / template clone | `brew install gh && gh auth login` |
| **Claude Code** | Agent orchestrator | [docs.claude.com/claude-code](https://docs.claude.com/claude-code) |
| *(optional)* `cadical` / `kissat` / `z3` | SAT / SMT solvers | `brew install cadical kissat z3` — on demand |
| *(optional)* `sage` | `polyrith` backend in Lean | `brew install --cask sage` — on demand |

Default Python dependencies (from `pyproject.toml`): `sympy`, `gmpy2`, `numpy`, `scipy`, `networkx`, `python-sat`, `mpmath`. Add more with `uv add <pkg>` and commit `pyproject.toml` + `uv.lock`.

---

## ❓ FAQ

### Why Lean 4?

Three reasons. **Mathlib coverage** — the Prime Number Theorem, the Polynomial Freiman–Ruzsa conjecture, the Liquid Tensor Experiment, and a huge swath of modern mathematics are already formalized; the harness's "browse-first, reuse-before-reprove" rule compounds. **LLM training-data abundance** — Mathlib has been on GitHub since well before any LLM cutoff, and recent neural-theorem-proving systems target Lean 4. **Toolchain stability** — `lean-toolchain` + `lake-manifest.json` give per-clone reproducibility that survives upstream churn.

### What if the harness can't make progress?

It writes `findings/deadend-<topic>-<date>.md` explaining the concrete obstacle, commits it, and switches to a different subgoal. The 30-minute cap per stuck subgoal is enforced by `/solve`. If a whole vector stalls, run `/vector retire` (or `/vector pivot` for a strategic shift).

### How does it avoid hallucinating proofs?

The Lean kernel is the trust root. `/solve` independently re-runs `lake build` and `#print axioms` rather than trusting the sub-agent's report. The critic agent reviews adversarially before any `[verified]` promotion — for high-stakes promotions, an entire **critic panel** (one critic per failure-mode lens) must clear it. Honest tagging (`[sketch]` / `[conditional]` / `[numerical]` / `[heuristic]`) is mandatory for anything that doesn't clear the axiom gate.

### Do the agents really run in parallel?

Yes — independent specialists run concurrently (and broad surveys / high-stakes reviews fan out into research-sweep and critic-panel **fleets**; see [Orchestration & parallelism](#orchestration--parallelism)). The one thing that is *not* parallelized is the Lean build: Lake has no inter-process build lock, so two `lake build`s in the shared `formal/.lake/` would race and corrupt each other rather than queue. Since a single `lake build` is already multi-core, the harness funnels all builds through one builder at a time and loses nothing — everything else (writing `.lean` files, read-only `lake env lean` checks, literature search, numerics) still runs concurrently around it.

### Can I edit the manuscript myself?

Yes. Commit your edits; `/solve` respects them on the next session. The style enforced by the harness is "publication-quality arXiv preprint" — no project scaffolding, no "Step 1 / Step 2" labels, no "Critic review PASS" remarks. If you edit, follow the same style (see `.claude/skills/solve/SKILL.md` § "Manuscript style").

### How do I switch strategy mid-project?

`/vector pivot`. The skill retires multiple vectors, records a strategic rationale in `findings/pivot-DATE.md`, and seeds replacement vectors in one ceremony commit.

### Why a single squashed initial commit on the template?

Every clone inherits the template at one point in time; dev history is irrelevant to that. A clean initial commit (`init: formalia`) gives every clone a tidy starting point.

---

## 📜 License

MIT — see [LICENSE](LICENSE).

---

<div align="center">

*The kernel decides.*

[Back to top](#top)

</div>

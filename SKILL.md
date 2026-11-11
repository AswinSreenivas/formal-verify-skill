---
name: formal-verify
description: Formally verify a piece of logic on demand, in any repo — a Python or TypeScript function, a rule buried in SQL/Spark, a race or ordering question ("can an abort kill a sibling request?"), a plain-English invariant, every function changed in the current diff, or every function changed in a GitHub PR (`--pr N`, read from the PR's head without switching branches) with a paste-ready report for the PR body. Claude drafts the rules, the user approves them, Claude writes a model (Dafny for per-input logic, TLA+ for event ordering) with file:line pointers to the real code, the checker verifies it, deliberately broken mutants must each be caught, and any counterexample is replayed against the real code. Use when the user says "/formal-verify", "formally verify X", "prove X", "model-check this race", "prove this always holds", "verify this logic", "check the diff", "formally verify PR 1234", or "verify this PR".
---

# formal-verify

Harness: `~/.claude/skills/formal-verify/fv` (Python 3.9 stdlib). Run it from inside the git checkout
that holds the code. It uses `git rev-parse --show-toplevel`.

The loop is the one from Anthropic's formal-verification work: **you don't trust Claude, you trust the
checker.** Only two things need human trust:
- the rule files, which the user approves;
- the checker itself (Dafny or TLC).

Everything Claude writes after that is checked, and the check itself must be shown able to fail.

```bash
F=~/.claude/skills/formal-verify/fv
$F doctor                     # dafny / java / tla2tools present? flags? free disk
$F selftest                   # negative controls: does fv still read Dafny/TLC output right? (~10 s)
$F targets [--base B]         # changed .py/.ts functions vs the default branch (+ uncommitted, untracked) and their proof state
$F snapshot <file>:<func>     # create/refresh the proof dir, pin a hash of the source
$F snapshot english:<slug> --statement "EOP(N) = BOP(N+1)"
$F skip <file>:<func> --note "plumbing: ..."   # record a triaged skip; goes stale when the code changes
$F approve <dir>              # ONLY after the user approves the rule files
$F check <dir>                # check the model, then run every mutant (each must FAIL)
$F mark <dir> bug|model-mismatch|unconfirmed|gave_up|skipped --note "..."
$F status                     # every proof for this repo; stale ones flagged
$F report [--base B | --all]  # markdown table for the current diff + a paste-ready Proof-of-Validation row
```

Every command except `doctor`, `selftest`, `approve` and `mark` takes `--pr N`. See **PR mode** below.
In zsh (the Bash tool's shell), write targets as `"${T}:func"` or as a literal path. `$T:a…` is read
as zsh's `:a` path modifier and mangles the target.

Proof dirs: `~/.claude/formal-proofs/<repo>/<file-slug>.<func>/`. `<repo>` is the main checkout's
name, so every worktree of a repo shares one store.

| File | Dafny | TLA+ |
|---|---|---|
| rules (trusted, user-approved) | `rule.dfy` | `rule.tla` + `rule.cfg` |
| model (Claude's) | `model.dfy` | `Model.tla` |
| mutants (deliberately broken models) | `mutant_<what>.dfy` | `mutant_<what>.tla` |
| pins | `rule.sha` (by `approve`), `source.sha` + `source.ref` (by `snapshot`) | same |
| output | `result.json` (incl. git commit), `last.log` | same |

A verdict counts only while **both** the code and the rule files still match their pins. `status`
flags `stale` (the code moved) and `rule-changed` (the rules were edited after approval).

## Procedure

### 1. Resolve targets

| Request | What to do |
|---|---|
| a function (`path:func`, or `path:Class.method`) | `fv snapshot` it |
| logic inside SQL or Spark | snapshot the Python/TS function that builds or runs it. The model is of the SQL semantics |
| an ordering / race question | snapshot the function where the steps happen, or `english:<slug>` if it spans services |
| a plain-English invariant | `fv snapshot english:<slug> --statement "..."` |
| no argument | `fv targets`. Skip every target already `fresh` + `proved`/`skipped`. Others go to step 2 |
| a PR (`--pr N`, `#N`, a PR URL, "this PR") | follow **PR mode** below |

Triage each remaining target in one line:
- **verifiable**: there is logic worth stating as rules (arithmetic, allocation, rounding,
  date/period math, state transitions, filters, ordering of concurrent steps);
- **skip — <reason>**: plumbing, IO, logging, wiring. Tell the user what was skipped and why, and
  record it with `fv skip <file>:<func> --note "<reason>"` so the report shows it as a decision,
  not a gap. Don't model plumbing.

### 2. Let the question choose the tool

| The question | Tool | Why |
|---|---|---|
| Is this function right **for every input**? (sums, bounds, rounding, NULL handling, date math) | **Dafny** | proves for all inputs, no bound |
| Can things happen **in a bad order**? (aborts, retries, two requests racing, cache fill vs invalidation, a read hitting a stale replica) | **TLA+ (TLC)** | explores every interleaving of steps in a small model |
| Both | two proof dirs, one per tool | — |

Tell the user which tool and why, in one line.

### 3. Rules first, then stop for approval

**Dafny.** `rule.dfy` is an **abstract module** that declares the signature and the rules as
`requires`/`ensures`, with no body. Every `ensures` line is one rule:

```dafny
abstract module Rule {
  // allocate_units: split `target` whole units across rows in proportion to `raw`
  method Allocate(raw: seq<real>, target: int) returns (shares: seq<int>)
    requires target >= 0
    requires forall i :: 0 <= i < |raw| ==> raw[i] >= 0.0
    requires |raw| > 0
    ensures |shares| == |raw|
    ensures Sum(shares) == target                 // nothing lost, nothing invented
    ensures forall i :: 0 <= i < |shares| ==> shares[i] >= 0
  function Sum(s: seq<int>): int { if |s| == 0 then 0 else s[0] + Sum(s[1..]) }
}
```

**TLA+.** `rule.tla` is `MODULE rule`, `EXTENDS Model`, and defines only the invariants/properties
over the model's variables. `rule.cfg` says `SPECIFICATION Spec`, lists the rules under
`INVARIANT`/`PROPERTY`, and fixes the bounds under `CONSTANTS` (e.g. 2 requests, 3 years). Each
listed name is one rule. The bounds are part of what the user approves: a pass holds only
within them.

```tla
---- MODULE rule ----
EXTENDS Model
\* aborting one request never cancels a sibling's in-flight write
NoSiblingKilled == \A r \in Requests : aborted[r] => \A s \in Requests \ {r} : ~cancelledBy[s][r]
====
```

Show the user each rule in plain English next to its formal line. Also show the input domain or
bounds explicitly (`requires` / `CONSTANTS`), because that is where a proof quietly gets narrowed.
**Wait for approval.** On approval, run `fv approve <dir>`. Never run `approve` on your own initiative.

Pick rules that would catch a real bug: conservation (sums), bounds, monotonicity, idempotence,
round-trips, "never touches X", "never ends in state Y", and agreement with a simpler reference
definition. Avoid rules that restate the implementation.

### 4. Model, true to the code

**Every part of the model points at the code it models.** Put a `file:line` comment on each
function, branch, action and step, e.g. `// billing.py:42` or `\* worker.ts:118`.
- `fv check` refuses a code-target model with no pointers, and warns on pointers past the end of
  the file.
- What the model treats as one atomic step must be one atomic step in the code. In TLA+, one
  action = one step that can't be interleaved: an `await` point, a lock section, a single SQL
  statement. Never merge two awaits into one action.
- Use this repo's names for variables and actions.

**Dafny:** `model.dfy` must `include "rule.dfy"` and refine the rule module, so the approved
`ensures` cannot be dropped or weakened:

```dafny
include "rule.dfy"
module Model refines Rule {
  method Allocate(raw: seq<real>, target: int) returns (shares: seq<int>) {
    // billing.py:42 — floor every raw share
    ...
  }
}
```

| Real code | Dafny model |
|---|---|
| pandas frames / lists | `seq<...>` |
| `NULL` / `NaN` / `None` | `Option<T>` or an explicit flag |
| `floor` / `round` | defined explicitly, e.g. Python `round` is banker's rounding and DuckDB `ROUND` is half-away-from-zero |
| `int` division | Python `//` floors toward −∞, Dafny `/` on `int` is Euclidean, JS `Math.trunc` truncates. Model the one the code uses. |
| IEEE floats | Dafny `real` is exact. Say so when the code depends on float behaviour. |

`fv check` refuses a Dafny model that uses `assume`, `{:axiom}`, `{:verify false}`, `{:extern}`,
`{:only}` or `decreases *`. Don't work around that.

**TLA+:** `Model.tla` is `MODULE Model` and defines the variables, `Init`, the actions, `Next` and
`Spec`. `fv` stages it next to the rule files in a temp dir and deletes TLC's state files after
every run.

### 5. Mutants: every rule must be seen failing

A check that has never failed proves nothing. For **each rule**, write at least one mutant: a copy of
the model with one realistic break that should violate that rule, named for the break. Examples:
- `mutant_no_topup.dfy`: drop the largest-remainder top-up;
- `mutant_abort_before_check.tla`: swap two steps;
- `mutant_skip_guard.tla`: remove a guard or a lock.

Header comment: which rule it targets and which code line it breaks. Mutants are copies of the
model, so they keep the same `include`/`MODULE Model` header.

`fv check` runs the model, then every mutant. Outcomes:

| Outcome | Meaning | Next |
|---|---|---|
| `proved` | model passes, every mutant is caught, every rule broken by some mutant | report |
| `proved-partial` | passes, but some rule was never seen failing | add a mutant for each listed rule |
| `mutant-survived` | a broken model still passes — the model or rule is too weak (vacuous) | fix the model, never the rule; if the rule is genuinely too weak, go back to step 3 and ask |
| `needs-mutants` | no mutants yet | write them |
| `failed` | the model itself fails | step 6 |

A mutant that never really got checked is `INVALID` and counts for nothing. Fix it. That covers a
parse or type error, a timeout, a Dafny warning such as a method left without a body, "0 verified",
and a TLA+ spec with no reachable states. The same conditions make the model itself `failed`, never
`proved`.

### 6. When the model fails

Run `fv check <dir>`:
- If it's a missing lemma or invariant, fix the **proof**.
- Change the **model** only if it doesn't match the real code (check the pointers).
- Never edit the rule files after approval. If a rule is wrong, go back to step 3 and ask again.
- **Hard cap: 5 attempts.** Then `fv mark <dir> gave_up --note "<what blocked it>"` and move on.

When it looks like a genuine violation, build the concrete case:
- Dafny: from its counterexample in `last.log`, or from the failing assertion.
- TLC: the trace in `last.log`, where `State 1..N` is the shortest bad sequence. **Tell it as numbered
  steps in the code's words**: "1. request A starts its write (worker.ts:118) 2. the user aborts request B …".

Then run the **real** code on that case, from a scratch file in the scratchpad (never edit the repo):

| Language | Replay |
|---|---|
| Python | import the function with the repo's venv if one exists, else `python3`. For heavy modules, copy the function body into the scratch file. |
| TypeScript | `npx tsx scratch.ts` |
| Self-contained SQL | DuckDB in Python: `CREATE TABLE` with the tiny input, run the SQL |
| An ordering trace | reproduce the interleaving with explicit awaits/barriers in a scratch script, if the code can run locally |
| Spark / live tables / IO / multi-service | can't replay |

| Replay result | Mark | Then |
|---|---|---|
| real code also breaks | `fv mark <dir> bug --note "<input or steps> → <wrong result>, expected <x>"` | report it with the exact input or steps |
| real code is fine | `fv mark <dir> model-mismatch` | the model was wrong: fix it and re-check. **Never report a mismatch as a bug.** |
| can't replay | `fv mark <dir> unconfirmed` | — |

Never fix the code yourself, and never weaken the model or rules to make a finding go away. Report
the bug; the user decides.

### 7. Report

Run `fv report` (or `fv report --pr N`) and relay a short table: target → tool → proved / bug (with
input or steps) / unconfirmed / gave_up / skipped (reason). Always include:
- its caveats line;
- for TLA+, the bounds (the report prints them on each proved TLA+ row).

`fv report` covers the functions the diff changes, plus `english:` proofs tied to this branch or
PR. `--all` covers every proof in the repo. The output is markdown: a `### Formal verification` section
for the PR body, then a "Proof of Validation" row in the `git-pr` template's `| command | result |`
shape. It exits 0 only when every target is `proved` or `skipped`, and lists the rest on stderr.

## PR mode

`--pr N` reads every source from PR N's head commit. `fv` runs `gh pr view N` for the base branch and
head SHA, then `git fetch origin pull/N/head <base>`. The working tree and the current branch are
never touched, so this works from any checkout, including the main one on a release branch.

1. `fv targets --pr N`: the PR's changed functions (merge-base..head, tests excluded) and their
   proof state.
2. Triage every row (step 1). `fv skip <file>:<func> --note "..." --pr N` for plumbing. For the rest,
   use `fv snapshot <file>:<func> --pr N` and steps 2–6, running `fv check <dir> --pr N`.
   Pointers are checked against the PR's version of the file.
3. An invariant that spans the PR (`english:<slug>`): `fv snapshot english:<slug> --statement "..."
   --pr N` ties it to the PR, so the PR report includes it.
4. `fv report --pr N`, then hand the user the markdown section and the Proof-of-Validation row for
   the PR body. Print only: never post to the PR or edit its body yourself.

Freshness is judged against the source you name. A proof made with `--pr N` is `fresh` under
`--pr N`, and may read `stale` or `source-missing` from a checkout on another branch. When the PR gets
new commits, re-run `fv report --pr N`: any proved function whose code moved shows as `stale` and
needs `snapshot` + `check` again.

## Limits (say these, don't hide them)

- **It proves the model, not the code.** Three things narrow that gap:
  - `file:line` pointers;
  - replay of counterexamples;
  - `source.sha`, which makes a proof go `stale` the moment the code changes.
- Only covers the approved rules, over the approved input domain (`requires`) or bounds (`CONSTANTS`).
- `fv targets` finds Python functions with `ast`, and TS functions with a header regex plus
  naive brace matching. Braces inside strings can mis-size a TS function, so eyeball the snippet
  if it looks off.
- **Cost.**
  - Dafny: `--cores 4`, 60 s per assertion, when the installed dafny supports those flags.
  - TLC: 4 workers, 4 GB heap, 600 s cap per run. If it hits the cap, shrink the bounds.
  - The real cost is tokens, which is why the attempt cap exists.

- `fv targets` / `fv report` without `--pr` diff against the remote's default branch (`origin/HEAD`,
  else `origin/main`). On a long-lived branch (e.g. a release branch), pass `--base`, or `--pr N`,
  or it lists hundreds of functions.
- `--pr N` sees only what is pushed to the PR. Uncommitted work is checked without `--pr`.

## Setup

`fv doctor`. Ask the user before installing anything. After installing or upgrading a checker, run
`fv selftest`. Until it passes, don't trust any `PROVED`: `fv` reads checker output with regexes,
and a new checker version can change that output.

| Tool | Install | Notes |
|---|---|---|
| Dafny | `brew install dafny` | pulls dotnet@8 + z3 |
| TLA+ | `brew install openjdk` (~200 MB), then put `tla2tools.jar` from github.com/tlaplus/tlaplus/releases at `~/.claude/skills/formal-verify/tools/tla2tools.jar` (or set `$TLA2TOOLS`) | macOS's `/usr/bin/java` is a stub with no runtime |

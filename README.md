# formal-verify

A [Claude Code](https://claude.com/claude-code) skill that formally verifies a piece of logic on
demand: a function, a rule hidden in SQL, a race between requests, or every function changed in your
current diff.

You don't have to trust Claude's "looks right". Claude writes a model and a proof, and a checker
verifies it:
- **[Dafny](https://dafny.org)** for "is this function correct for every input?";
- **[TLA+](https://lamport.azurewebsites.net/tla/tla.html)** for "can these steps happen in a bad order?".

Inspired by Anthropic's
[LLM-scaled formal verification](https://www.anthropic.com/research/formal-verification) work, which
uses the same loop: the model writes the proof, and a small trusted checker decides.

## How a run goes

1. **Rules.** Claude drafts 2–4 rules in plain English plus formal form, e.g. "the shares always sum
   to the target". You approve them. They are the only part you have to trust.
2. **Model.** Claude models the code. Every part carries a `file:line` pointer to the real code.
3. **Check.** Dafny or TLC verifies the model against the approved rules.
4. **Must-fail check.** For each rule, Claude writes a deliberately broken copy of the model
   (a "mutant"). The checker must catch every one, and between them they must break every rule. A
   rule never seen failing proves nothing.
5. **Counterexamples are replayed against the real code.**
   - The real code also breaks → it's reported as a **bug**.
   - The real code is fine → the model was wrong. It's fixed, never reported.

The harness never edits your code, and never weakens a rule to make a finding go away.

Proofs are saved in `~/.claude/formal-proofs/<repo>/`. A proof goes **stale** automatically when the
code it covers changes, or when the approved rules are edited.

## Install

```bash
git clone https://github.com/AswinSreenivas/formal-verify-skill ~/.claude/skills/formal-verify
~/.claude/skills/formal-verify/fv doctor     # shows what's missing
~/.claude/skills/formal-verify/fv selftest   # after installing a checker: negative controls, ~10 s
```

Checkers (install whichever you need):

| Checker | Install | Notes |
|---|---|---|
| Dafny | `brew install dafny` | pulls in .NET 8 + Z3 |
| TLA+ | `brew install openjdk` | then put `tla2tools.jar` from the [TLA+ releases](https://github.com/tlaplus/tlaplus/releases) at `~/.claude/skills/formal-verify/tools/tla2tools.jar` (or set `$TLA2TOOLS`) |

`fv` itself is one Python 3.9+ script with no dependencies. Supported languages:
- finding changed functions: Python and TypeScript;
- modelling: any language, because the model describes the logic, not the syntax.

## Use

In Claude Code, inside your repo:

```
/formal-verify src/billing.py:allocate_units
/formal-verify "a cancelled request never cancels another request's write"
/formal-verify                      # every function changed vs the default branch
/formal-verify --pr 1234            # every function changed in a GitHub PR
```

### PR mode

`--pr N` reads the code from the PR's head commit, via `gh` and `git fetch`. It never checks out the
PR or touches your working tree, so it works from any checkout. `fv report --pr N` prints a markdown
section for the PR body: one row per changed function (proved / skipped with a reason / bug / needs
proof), plus a one-line Proof-of-Validation summary. It exits non-zero until every changed function
is proved or deliberately skipped. Nothing is posted to GitHub.

## Limits

- **It proves the model, not the code.** Three things narrow the gap:
  - `file:line` pointers;
  - replaying counterexamples;
  - stale detection.
- A proof only covers the approved rules, over the approved input range or (TLA+) size bounds.
- Dafny's `real` is exact math, not IEEE floats.
- `fv` reads checker output with regexes. `fv selftest` runs negative controls covering a correct
  model, a broken one, an empty one, a vacuous one, an unsatisfiable `Init`, and an unparseable
  mutant, and checks that each is classified right. Tested with Dafny 4.11 and TLA+ tools 1.7.4.
  Re-run it after upgrading either one, and open an issue if a control fails.

## Credits

The must-fail mutants, choosing the tool by the question, and code pointers are ideas adapted from
Cursor's [`create-formal-verification-skill`](https://github.com/cursor/plugins/blob/main/pstack/skills/create-formal-verification-skill/SKILL.md).

## License

MIT

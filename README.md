# auto-decide

A Claude Code skill that autonomously picks the best implementation option and keeps building — instead of presenting choices and waiting for the user to decide.

## How It Works

When Claude encounters multiple implementation paths (e.g., "Option A vs Option B"), this skill replaces the default "here are your options, what do you think?" behavior with a principled decision:

1. **Anchor on the original request** — distill what the user actually wants
2. **Evaluate each option** internally against three criteria:
   - **Feature completeness** (heaviest weight) — every requested feature must be present
   - **Fidelity to intent** — no quiet substitutions or reinterpretations
   - **Comprehensiveness** (tiebreaker) — handles more real-world scenarios
3. **Decide in 1-2 sentences and keep building** — no hedging, no "let me know if you'd prefer otherwise"

### Quick Reference

| Situation | Action |
|-----------|--------|
| One option drops a feature | Eliminate it — feature completeness wins |
| Options are equally complete | Pick the more comprehensive one |
| Options are genuinely equivalent | Pick the simpler one, state they were equivalent |
| All options skip something | Pick the one that skips the least important feature, flag it |
| Option contradicts user constraint | Eliminate immediately — constraints are non-negotiable |
| Original request is ambiguous | Use the most generous reasonable interpretation |

## Installation

```bash
npx skills add mbaranovski/auto-decide
```

Or manually copy `SKILL.md` into your Claude Code skills directory.

## Evaluation Suite

The `evals/` directory contains 9 test scenarios covering:

- **Deferred features** — options that push requested features to "phase 2" (evals 1, 2, 3, 7, 9)
- **Constraint violations** — options that ignore explicit user requirements (eval 4)
- **Quiet substitutions** — options that swap the requested design for something "simpler" (evals 5, 9)
- **Fidelity degradation** — options that technically deliver features but at reduced quality (evals 1, 7)
- **Tie-breaking** — genuinely equivalent options where the skill must justify its pick (evals 6, 8)

Each eval includes a prompt, expected output, behavioral assertions, and human-reviewed results in `evals/eval-{N}/outputs/`.

## What This Skill Is NOT

- **Not a license to over-engineer.** "Comprehensive" means handling real scenarios, not adding abstractions for hypothetical ones.
- **Not a way to ignore user preferences.** Stated preferences are constraints, not suggestions.
- **Not always the "biggest" option.** Unnecessary complexity isn't comprehensive — it's bloated.

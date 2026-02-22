# User Notes — Eval 9

## What Happened

The auto-decide skill was given a three-way choice for implementing a user authentication system. The user's original request explicitly named four features: email/password login, OAuth (Google + GitHub), password reset, and refresh tokens.

## Decision Made

**Option B** was selected — full implementation of all features exactly as requested.

## Why This Was the Right Call

This eval was designed so that the correct answer is unambiguous under the skill's framework:

- **Option A** substitutes long-lived JWTs for the explicitly requested refresh tokens. The skill specifically flags this pattern: "Does it quietly swap in something easier to build?" Option A does exactly that. It is a common real-world temptation — long-lived JWTs are simpler and "good enough" — but the skill correctly penalizes it because the user asked for refresh tokens by name.

- **Option C** defers two of four features to later phases. The skill's core principle states "nothing is deferred to a future iteration." Option C violates this directly.

- **Option B** delivers everything requested, exactly as requested. No substitution, no deferral.

## What This Eval Tests

1. **Fidelity to intent over simplicity** — Option A is the "pragmatic developer" choice. Many engineers would pick it. The skill should not, because the user was specific about refresh tokens.

2. **Resistance to phased delivery** — Option C is the "agile" or "incremental" choice. It sounds reasonable ("get the base working first"). The skill should reject it because the user asked for everything now.

3. **No hedging after the decision** — The skill requires a clean 1-2 sentence decision and then immediate continuation. No "let me know if you'd prefer otherwise." The response should transition directly into implementation.

## Observations

- The simulated response correctly identified Option B as the winner, stated the decision concisely, and moved into implementation without hedging.
- The reasoning correctly identified that Option A's substitution of long-lived JWTs for refresh tokens was a fidelity violation, not just a minor simplification.
- The implementation plan that followed the decision was concrete and covered all four features with appropriate detail (data model, endpoints, token rotation with reuse detection).
- No issues were encountered. The skill's decision procedure handled this scenario cleanly.

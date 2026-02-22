# User Notes -- Eval 1: Real-Time Chat Application

## What Was Tested

Whether the auto-decide skill correctly applies its decision procedure to select the option that most completely and faithfully delivers the user's original request, given three options with varying levels of feature completeness and real-time fidelity.

## Observed Behavior

The skill's decision procedure produced a clear, well-grounded result:

- **Option C eliminated immediately.** The skill's heaviest-weighted criterion is feature completeness, with an explicit rule that nothing should be deferred to "phase 2." Option C defers two of three requested features. This was a clean, unambiguous elimination.
- **Option B eliminated on fidelity grounds.** Option B technically addresses all three features, but the typing indicator and read receipt implementations use 2-second REST polling instead of real-time delivery. The user described the entire application as "real-time," and the skill's fidelity criterion flagged this as "quietly swapping in something easier to build." This is the more interesting evaluation -- it required looking past a surface-level feature checklist to assess whether the *quality* of delivery matched the user's intent.
- **Option A selected.** It was the only option that delivered all three features with the real-time behavior the user requested, over a single transport mechanism.

## Skill Strengths Demonstrated

1. **Weighted criteria create a clear decision path.** Feature completeness as the primary weight made Option C trivial to eliminate. Fidelity as the secondary weight correctly distinguished between Options A and B, which both nominally deliver all features but at different quality levels. The hierarchy prevented false equivalence.
2. **The "phase 2" prohibition is well-calibrated.** This rule directly caught Option C's strategy of relabeling user-requested features as "nice to have." Without this rule, a model might accept the reframing and choose Option C for its simplicity.
3. **The "continue working" instruction worked.** After the 1-2 sentence decision, the response moved into a concrete implementation plan (message protocol, server architecture, client-side logic, message flow). No deliberation was exposed to the user.
4. **No hedging language.** The response did not include "let me know if you'd prefer otherwise," "we could revisit this," or any other soft reversals. The decision was stated and work continued.

## Potential Concerns

1. **The response did not literally start writing code.** The skill says to "immediately proceed with implementation," and the response provided an implementation *plan* rather than actual source code. This is defensible -- no project structure, language, or framework was specified, so generating code would require assumptions the user hasn't made. But in a session where a codebase exists and the language is obvious, the expected behavior would be to start generating files, not describe what those files would contain. Future evals should test this in a codebase context.
2. **Decision statement was slightly more than 1-2 sentences.** The decision itself was two sentences, but the rationale included brief parenthetical explanations of why the other options lost. This adds clarity for the user but stretches beyond the skill's "1-2 sentence decision statement" instruction. It did not devolve into a full option-by-option comparison, so the spirit of the skill was preserved, but it is worth monitoring in future evals.
3. **No user-stated constraints to test.** All three options were purely generated alternatives with no user-stated technology preferences or constraints. The skill's rule about user constraints being "non-negotiable" was not exercised. A future eval should include a scenario where the user has stated a preference (e.g., "I want to use Firebase") and one option contradicts it.

## Verdict

The skill performed as designed. The correct option was selected, the reasoning was sound, the decision was stated concisely, and implementation followed without pause. This is a clean pass.

The eval is most valuable as a baseline -- it tests the straightforward case where feature completeness creates a clear winner. More challenging evals should test scenarios where all options deliver all features (forcing comprehensiveness to break the tie), where the user has stated constraints that conflict with the "best" option, or where the original request is ambiguous.

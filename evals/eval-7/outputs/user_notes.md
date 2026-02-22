# User Notes -- Eval 7

## What Happened

The auto-decide skill was given four options for building an inventory management system. The original request specified four features: real-time stock tracking, low-stock alerts, purchase order generation, and a reporting dashboard with stock trends over time.

## Decision Made

**Option A** was selected -- full implementation of all four features with WebSocket, email alerts, PDF PO export, and Chart.js trend dashboard.

## Why This Decision

The skill's core principle is "choose the option that delivers the original request most completely." This made the evaluation clear:

- **Option B** was eliminated immediately -- it defers two of the four requested features to "phase 2," which the skill explicitly penalizes ("nothing is deferred to a future iteration").
- **Option C** was eliminated for dropping "trends over time" from the dashboard, which was an explicit part of the original request. The skill flags this as swapping in "something easier to build."
- **Option D** delivered all four features but simplified several (polling instead of real-time, CSV instead of PDF, in-app only instead of email alerts). It was the only real competitor to Option A.
- **Option A** won the A-vs-D comparison on comprehensiveness: WebSocket doesn't need rework later, PDF is the business standard for POs, and email alerts work even when users aren't in the app.

## Skill Behavior Observations

1. **The "no phase 2" rule is very effective.** It immediately eliminated Option B without needing deep analysis. This is a strong guardrail that prevents the skill from choosing the "easy path" of deferral.

2. **Fidelity checking caught Option C.** The user said "trends over time" and Option C dropped the "over time" part. The skill correctly identified this as a fidelity violation rather than a minor simplification.

3. **The comprehensiveness tiebreaker resolved A vs D cleanly.** Both had 4/4 feature completeness, but A needed less future rework. The skill's tiebreaker criteria ("needs less rework as requirements grow") gave a clear winner.

4. **No over-engineering concern with Option A.** The skill's "not a license to over-engineer" guardrail was considered, but Option A's components all map directly to requested features -- nothing is speculative or hypothetical.

5. **Response format was correct.** Decision stated in 2 sentences, no hedging, no confirmation request, immediate continuation with implementation details.

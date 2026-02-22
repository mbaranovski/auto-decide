# User Notes -- Eval 2: Auto-Decide Skill

## What Was Tested

A three-option architectural decision for an admin dashboard with filtering, sorting, and CSV export. The options ranged from a simple client-side approach (A), to a server-side approach that defers CSV export (B), to a fully server-side approach with smart CSV export handling (C). This tests the skill's ability to eliminate an option that defers a feature and then use the comprehensiveness tiebreaker between two feature-complete options.

## Decision Made

**Option C** was selected: server-side filtering, sorting, and pagination with dual-mode CSV export (immediate download for small datasets, async email for large ones).

## How the Skill Performed

The skill worked as designed. Key observations:

1. **Feature completeness correctly dominated the decision.** The skill's heaviest weight is feature completeness, and Option B explicitly defers CSV export -- one of three named features. The skill correctly eliminated it without hesitation. This is the skill's primary value: it prevents the common failure mode where Claude picks the "simpler" option that quietly drops a feature.

2. **The comprehensiveness tiebreaker worked correctly.** Between Options A and C (both 3/3 on features), the skill correctly used comprehensiveness as the tiebreaker. An admin dashboard with a 10k row ceiling (Option A) is not a viable real-world solution. Option C handles both small and large datasets without rework.

3. **No over-engineering.** The skill correctly identified that Option C's dual-mode export (immediate download vs. async email) is a practical pattern, not over-engineering. The "What This Skill Is NOT" guardrail held -- the response did not add unnecessary abstractions or speculative features beyond what was needed.

4. **Decision stated concisely and work continued immediately.** The decision was expressed in two sentences. No hedging, no "let me know if you prefer otherwise." The response moved straight into a detailed implementation plan with concrete API designs.

5. **Implementation plan was substantive.** The response did not just name the decision -- it laid out a five-part implementation plan with specific API shapes, filter operators, pagination parameters, and the CSV threshold logic. This demonstrates the skill's intent: make the decision and keep building.

## Decision Quality Assessment

**The decision was correct.** Option C is the right call for this prompt. The reasoning chain was sound:
- B is out because it defers a feature (CSV export) that was explicitly requested.
- A vs C: both complete, but C is more comprehensive and handles the real-world scaling concern that matters for admin dashboards.
- C is not the "biggest" option for the sake of being big -- it solves a genuine problem (large CSV exports blocking or timing out) with a standard approach (async job + email notification).

## Comparison to Previous Run

The previous run of this eval also selected Option C with similar reasoning. This run produced a more detailed implementation plan (five sections with specific API designs vs. four bullet points previously). The decision statement was slightly more pointed in calling out why A and B failed. The core reasoning was consistent across both runs, which is a positive signal for the skill's reliability.

## Edge Cases Worth Noting

- This eval had a clear winner. A harder test would be one where all three options deliver all features but with different trade-offs in complexity, performance, and maintainability.
- Option B is almost a trap -- it sounds pragmatic ("we'll add CSV in a follow-up PR"), and a default Claude response might favor it for being "practical." The skill's explicit rule against deferring features catches it cleanly.
- The "not over-engineering" guardrail was relevant but not tested under pressure here. A better stress test would include an Option D that delivers all features but adds unnecessary layers of abstraction (event sourcing, CQRS, etc.) to see if the skill correctly rejects it.

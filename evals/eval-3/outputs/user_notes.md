# User Notes -- Eval 3

## What Was Tested

The auto-decide skill was given four architecture options for a notification system. The original request has two explicit, unambiguous requirements:

1. Three notification channels (email, push, in-app)
2. User preferences for each channel

The four options probe different failure modes:

- **Option A:** Fully complete, well-architected (expected correct answer)
- **Option B:** Incomplete -- drops 2 of 3 channels, defers them to later
- **Option C:** Fully complete but less extensible (plausible distractor)
- **Option D:** Drops user preferences entirely, defers to next sprint

## Decision Made

**Option A** was selected: unified notification service with a plugin architecture and a dedicated preferences table.

## Why This Decision Is Correct

### Options B and D were correctly eliminated

Both defer explicitly stated requirements. The skill's heaviest criterion is feature completeness ("An option that skips a feature almost always loses, even if it's simpler"), which correctly rules these out. This is the most important behavior to verify -- the skill must not be seduced by simplicity (Option B) or architectural elegance minus a feature (Option D).

Option D is a particularly good test case because it has the best architecture (plugin system) but skips an explicit requirement. A naive "pick the most elegant" heuristic would choose D. The skill correctly prioritizes completeness over elegance.

### Option A was correctly chosen over Option C

Both A and C deliver all stated features. The tiebreaker in the skill is comprehensiveness: "which handles more real-world scenarios and needs less rework as requirements grow." Option A's plugin architecture and dedicated preferences table are more maintainable than Option C's if/else routing and JSON column, without crossing into over-engineering. The skill's guidance that "comprehensive" does not mean "biggest" is respected -- Option A's abstractions each serve a concrete purpose.

## Response Format Observations

- Decision stated in two sentences at the top -- matches the skill's "1-2 sentence" requirement.
- No hedging or "let me know if you'd prefer otherwise" -- matches the "No hedging" directive.
- Immediately continues into implementation plan -- matches "then immediately proceed with implementation."
- Internal analysis (the option-by-option evaluation) is not visible in the response -- matches "Do this analysis internally."
- The implementation plan is detailed enough to start building (schema, interface, service, API, sequence).

## Potential Edge Cases Worth Noting

- If the user had expressed a preference for simplicity or said "keep it simple for now," Option C might become defensible. The skill says "user constraints are non-negotiable." In this eval, no such constraint was stated.
- If the user had said "MVP first," Option B could be defensible. But "MVP" was not mentioned, so the skill correctly optimizes for the full request.
- Option C is the closest competitor and a good distractor. A weaker model might choose C because it is "simpler" without properly applying the comprehensiveness tiebreaker.

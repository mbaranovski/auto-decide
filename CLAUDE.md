# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Claude Code skill** called `auto-decide` — not a traditional software project. It defines a decision-making skill that enables Claude to autonomously choose between implementation options instead of asking the user to decide.

The repository contains:
- `SKILL.md` — The skill definition (YAML frontmatter + Markdown specification)
- `evals/` — Evaluation suite with 9 test scenarios and their results

## Skill Architecture

The skill replaces Claude's default "present options and ask" behavior with a principled decision framework:

1. **Feature completeness** (heaviest weight) — Does the option deliver every requested feature?
2. **Fidelity to intent** — Does it preserve what was actually asked for, without quiet substitutions?
3. **Comprehensiveness** (tiebreaker) — Which handles more real-world scenarios?

User-stated constraints are non-negotiable and eliminate options before evaluation begins. When options are genuinely equivalent, pick the simpler one and state they were equivalent.

## Evaluation System

Evals live in `evals/evals.json` — a JSON file with 9 structured test cases. Each eval has:
- `prompt`: A decision scenario with multiple options
- `expected_output`: The correct decision with reasoning
- `assertions`: 6-8 specific behavioral checks

Each eval has a corresponding `evals/eval-{N}/outputs/` directory containing:
- `response.md` — Claude's response to the prompt
- `transcript.md` — Full interaction transcript
- `user_notes.md` — Human evaluator's assessment

Evals are run manually by invoking Claude with each prompt and comparing against assertions. There is no automated test runner.

## Key Design Decisions

- `model: opus` — Skill requires Opus-level reasoning
- `disable-model-invocation: true` — Skill does not invoke sub-models
- `user-invokable: true` — Can be triggered directly by users
- `compatibility: planning, brainstorming, coding` — Integrates with other skill phases

## What the Evals Test

The 9 evals cover distinct decision patterns: deferring features to "phase 2" (evals 1-3, 7, 9), violating explicit user constraints (eval 4), quiet algorithm/design substitution (evals 5, 9), fidelity degradation (evals 1, 7), equivalent-option tie-breaking (evals 6, 8), and the "genuinely equivalent" edge case (eval 8).

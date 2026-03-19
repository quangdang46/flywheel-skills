---
name: flywheel-planner
description: >
  Guide the multi-model planning phase of the Agentic Coding Flywheel methodology.
  Use this skill whenever the user wants to: start planning a new software project,
  create a comprehensive markdown plan, use multiple AI models to synthesize the best
  architecture, or iteratively refine a plan before writing any code. Trigger on phrases
  like "plan this project", "help me design", "create a markdown plan", "multi-model
  planning", "synthesize plans", "architecture for", or any request to think through
  a project before implementation. This skill should also trigger when the user describes
  a project idea and wants to figure out how to build it — even if they don't explicitly
  say "plan".
---

# Flywheel Planner

Guide the user through the multi-model planning phase of the Agentic Coding Flywheel.
The core insight: **85% of the work is planning.** Global reasoning is cheapest while
the whole system still fits in a context window.

**Three reasoning spaces — always know which one you're in:**
- **Plan space:** Architecture, features, workflows. Whole system fits in context. Fix cost: 1x.
- **Bead space:** Executable task structure. Fix cost: 5x.
- **Code space:** Implementation. Fix cost: 25x.

Stay in plan space until the plan is genuinely stable.

---

## Step 0: Foundation Bundle (before writing anything)

Confirm these exist before starting the plan:

- [ ] Tech stack decision (if unclear → do a deep research round first)
- [ ] AGENTS.md bootstrapped from a known-good template
- [ ] Product intent and user workflows clearly articulated
- [ ] Best-practices guides for the chosen stack

**Rust CLI stack:** `crates/<name>/`, no `src/` at root, workspace Cargo.toml, ripgrep/fd/bat pattern.
**Web app stack:** TypeScript, Next.js, React, Tailwind, Supabase; Rust/WASM for perf-critical parts.

If the stack isn't obvious, suggest a research round before planning.

---

## Step 1: Write the Initial Plan

### Who writes it
The user doesn't need to write it themselves. Start from a rough stream-of-thought description
of the concept, then let GPT Pro (Extended Reasoning) flesh it out into a comprehensive
markdown plan. Claude Opus web is also good for this.

### What to explain to the model
- The concept and end goal
- User-visible workflows (how users actually interact with the system)
- Why it matters — the model does better when it understands intent, not just spec
- Any known constraints (perf, security, deployment model)

### What a first plan should cover
Not "build a notes app" but the actual system:
- What users can do and how
- What happens on failure paths
- What admins or operators need
- What auth/access model applies
- What test coverage proves it works

### Prompt the user to provide
Help them articulate their concept clearly. Ask:
1. What does this software DO from the user's perspective?
2. What are the 2–3 most important workflows?
3. What makes it different from existing solutions?
4. Any hard constraints (language, platform, latency, etc.)?

---

## Step 2: Competing Model Plans

After the initial plan exists, instruct the user to ask competing models to design the
same project independently:

- **GPT Pro** (Extended Reasoning) — system-wide coherence, best final arbiter
- **Claude Opus** (web app) — execution detail, sharp structural edits
- **Gemini** (Deep Think) — alternative framings, missed edge cases
- **Grok Heavy** — counterintuitive options, pressure-testing assumptions

Each model has different architectural tastes and blind spots. Running the same project
through all four is the cheapest way to buy architectural robustness.

Reference: The CASS Memory System competing plans are publicly visible at
`github.com/Dicklesworthstone/cass_memory_system/tree/main/competing_proposal_plans`

---

## Step 3: Best-of-All-Worlds Synthesis

After collecting competing plans, use this prompt in GPT Pro:

```
I asked 3 competing LLMs to do the exact same thing and they came up with pretty
different plans which you can read below. I want you to REALLY carefully analyze
their plans with an open mind and be intellectually honest about what they did
that's better than your plan. Then I want you to come up with the best possible
revisions to your plan (you should simply update your existing document for your
original plan with the revisions) that artfully and skillfully blends the "best
of all worlds" to create a true, ultimate, superior hybrid version of the plan
that best achieves our stated goals and will work the best in real-world practice
to solve the problems we are facing and our overarching goals while ensuring the
extreme success of the enterprise as best as possible; you should provide me with
a complete series of git-diff style changes to your original plan to turn it into
the new, enhanced, much longer and detailed plan that integrates the best of all
the plans with every good idea included:
[PASTE COMPETING PLANS]
```

Then paste GPT Pro's output into Claude Code to integrate:

```
OK, now integrate these revisions to the markdown plan in-place; use ultrathink.
Be meticulous. At the end, tell me which changes you wholeheartedly agree with,
which you somewhat agree with, and which you disagree with.
[PASTE SYNTHESIS OUTPUT]
```

The "wholeheartedly / somewhat / disagree" framing forces the model to evaluate each
revision on a gradient — you get signal about which changes need human review.

---

## Step 4: Iterative Refinement

Paste the current plan into a **fresh** GPT Pro conversation (fresh = no anchoring on
prior output). Repeat 4–5 rounds:

```
Carefully review this entire plan for me and come up with your best revisions in
terms of better architecture, new features, changed features, etc. to make it
better, more robust/reliable, more performant, more compelling/useful, etc.
For each proposed change, give me your detailed analysis and rationale/justification
for why it would make the project better along with the git-diff style changes
relative to the original markdown plan shown below:
[PASTE CURRENT PLAN]
```

### The "Lie to Them" technique
Models stop searching after ~20–25 issues. To force exhaustive search:

```
Do this again, and actually be super super careful: can you please check over the
plan again and compare it to all that feedback I gave you? I am positive that you
missed or screwed up at least 80 elements of that complex feedback.
```

Claiming 80+ errors forces the model to keep searching past where it would normally stop.

### When to stop refining
Stay in plan refinement if:
- Whole-workflow questions are still moving around
- Major architecture debates are still open
- Fresh models keep finding substantial missing features or constraints

Switch to beads when:
- The plan feels stable
- Remaining improvements are about execution structure and testing, not what the system IS
- Suggestions have become incremental rather than fundamental

Plans created this way routinely reach **3,000–6,000+ lines**. This is correct. They
are not slop — they are the result of countless iterations across multiple frontier models.

---

## Step 5: Adding Major Features to Existing Projects

For bounded improvements to an existing codebase, use the **Idea-Wizard** pipeline:

1. Read AGENTS.md and list existing beads (`br list --json`) — prevents duplicates
2. Generate 30 ideas, winnow to 5 with justification
3. Prompt: "ok and your next best 10 and why" — produces ideas 6–15
4. Human selects which to pursue
5. Convert selected ideas to beads (hand off to `flywheel-beads` skill)
6. Polish 4–5 times (same as any bead set)

```
Come up with 30 ideas for improvements, enhancements, new features, or fixes for
this project. Then winnow to your VERY best 5 and explain why each is valuable.
```

30→5 winnowing produces far better results than asking for 5 directly. The winnowing
forces critical evaluation that produces genuinely strong ideas.

---

## Validation Gate: Plan is Ready for Beads

Before handing off to the `flywheel-beads` skill, confirm:

- [ ] Plan covers all user workflows end-to-end
- [ ] Architecture decisions are made (not deferred)
- [ ] Failure paths and edge cases are documented
- [ ] Testing expectations are stated
- [ ] Major tradeoffs are explained with rationale
- [ ] Fresh model refinement rounds are returning only incremental suggestions

If any gate fails, keep refining. Do not move to beads with open architectural questions.

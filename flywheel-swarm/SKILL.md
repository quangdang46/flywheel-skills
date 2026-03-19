---
name: flywheel-swarm
description: >
  Launch and manage a coordinated AI agent swarm using the Agentic Coding Flywheel
  methodology. Use this skill whenever the user wants to: start a multi-agent coding
  session, orchestrate Claude Code + Codex + Gemini agents in parallel, manage agent
  coordination via Agent Mail and bv, handle stuck agents or compaction, run review
  and hardening rounds, or tend an active swarm. Trigger on phrases like "launch agents",
  "start the swarm", "agent coordination", "agent mail", "multiple agents", "parallel
  agents", "marching orders", "swarm is stuck", "review round", "cross-agent
  review", "harden the code", or any mention of running multiple coding agents simultaneously.
---

# Flywheel Swarm

Launch and tend a coordinated swarm of coding agents using br (beads), bv (graph routing),
and Agent Mail (coordination). This phase is deliberately mechanical — the hard thinking
already happened during planning and bead polishing.

**Human role:** Clockwork deity. Design the machine, set it running, manage it. The cognitive
work is done. Now you tend, not think.

---

## Coordination Triangle

Three tools — remove any one and the swarm loses determinism:

| Tool | Role | Without It |
|------|------|-----------|
| `br` | Durable task state — what agents execute | Agents have no structured work |
| `bv` | Graph-theory compass — which task unlocks most | Agents choose randomly |
| Agent Mail | Communication — coordination without spam | Agents step on each other |

---

## Step 1: Pre-Launch Checklist

- [ ] AGENTS.md exists and is complete (see required rules below)
- [ ] Beads are polished and validated (`bv --robot-insights` shows clean graph)
- [ ] Agent Mail project is initialized
- [ ] bv is installed and responsive
- [ ] DCG (Destructive Command Guard) is active

---

## Step 2: Launch the Swarm


### Staggered Start (critical — prevents thundering herd)
Start agents **30+ seconds apart**. Simultaneous starts cause agents to pile onto the
same frontier beads. Staggered starts let each agent claim distinct work naturally.

For Codex specifically: send Enter twice after pasting long prompts (input buffer quirk).

### Scale by Open Beads
| Open Beads | Claude (cc) | Codex (cod) | Gemini (gmi) |
|-----------|-------------|-------------|--------------|
| 400+ | 4 | 4 | 2 |
| 100–399 | 3 | 3 | 2 |
| <100 | 1 | 1 | 1 |

### Model Selection by Phase
| Phase | Best Model | Reason |
|-------|-----------|--------|
| Implementation | Claude Code (Opus) | Best for architecture + complex reasoning |
| Fast iteration | Codex | Complementary strengths, good at tests |
| Review duty | Gemini | Different perspective, good at finding issues |
| Final verification | Codex (GPT high reasoning) | Different model catches different bugs |

---

## Step 3: Marching Orders Prompt

Send to every agent after launch:

```
First read ALL of the AGENTS.md file and README.md file super carefully and understand
ALL of both! Then use your code investigation agent mode to fully understand the code,
and technical architecture and purpose of the project. Then register with MCP Agent Mail
and introduce yourself to the other agents.
Be sure to check your agent mail and to promptly respond if needed to any messages; then
proceed meticulously with your next assigned beads, working on the tasks systematically
and meticulously and tracking your progress via beads and agent mail messages.
Don't get stuck in "communication purgatory" where nothing is getting done; be proactive
about starting tasks that need to be done, but inform your fellow agents via messages when
you do so and mark beads appropriately.
When you're not sure what to do next, use the bv tool mentioned in AGENTS.md to prioritize
the best beads to work on next; pick the next one that you can usefully work on and get
started. Make sure to acknowledge all communication requests from other agents and that you
are aware of all active agents and their names. Use ultrathink.
```

The vagueness is intentional — specifics come from AGENTS.md and the beads themselves.
The same prompt works across every project.

---

## Step 4: AGENTS.md Required Rules

Every AGENTS.md must include:

1. **Rule 0 — Override:** Human instructions override everything
2. **Rule 1 — No deletion:** Never delete files without explicit permission
3. **No destructive git:** `git reset --hard`, `git clean -fd`, `rm -rf` are forbidden
4. **Branch policy:** All work on `main`, never `master`
5. **No script edits:** Always make code changes manually, not via scripts
6. **No file proliferation:** No `mainV2.rs`, `main_improved.rs` variants
7. **Compiler after changes:** Always verify no errors (`cargo check --all-targets`)
8. **Multi-agent awareness:** Never stash, revert, or overwrite other agents' changes
9. **Post-compaction:** After any compaction, reread AGENTS.md before continuing

Include tool blurbs for: `br`, `bv`, Agent Mail, `cass`/`cm`, `ubs`, `dcg`.
More content in AGENTS.md = more frequent compactions, but saves mistakes. Worth it.

---

## Step 5: bv Routing

Agents must use bv before picking any task:

```bash
bv --robot-triage        # Full recommendations — use this first
bv --robot-next          # Single top pick + exact claim command
bv --robot-plan          # Parallel execution tracks with unblock lists
```

**Priority patterns:**
| PageRank | Betweenness | Meaning | Action |
|----------|-------------|---------|--------|
| High | High | Critical bottleneck | Fix this before anything else |
| High | Low | Foundation piece | Important, not currently blocking |
| Low | High | Unexpected chokepoint | Investigate why this is a bridge |
| Low | Low | Leaf work | Parallelize freely |

**Only use `--robot-*` flags.** Bare `bv` opens interactive TUI and blocks the session.

---

## Step 6: Agent Workflow Per Bead

Each agent follows this sequence for every bead:

1. `bv --robot-triage` → identify highest-impact available bead
2. `br ready --json` → confirm it's unblocked
3. Reserve relevant files via Agent Mail file reservation
4. Announce claim in Agent Mail thread `[br-###]`
5. `br update <id> --status in_progress`
6. Implement + test locally
7. `cargo check --all-targets` (Rust) or equivalent
8. Self-review with fresh eyes (see review section)
9. `br close <id> --reason "Completed"`
10. Commit + push
11. Release file reservations
12. Back to step 1

---

## Step 7: Human Cadence (every 10–30 minutes)

1. **Check progress:** `br list --status in_progress --json` or `bv --robot-triage`
2. **Handle compactions:** Send "Reread AGENTS.md so it's still fresh in your mind."
3. **Periodic reviews:** Pick 1–2 agents that finished a bead, send cross-agent review prompt
4. **Organized commits:** Every 1–2h, designate one agent for commits
5. **New issues:** Create beads for unanticipated problems, don't improvise in-session

**Post-Compaction Reset (most common intervention):**
```
Reread AGENTS.md so it's still fresh in your mind.
```

---

## Step 8: Review Rounds

### Per-Bead Self-Review (after each bead)
```
Great, now I want you to carefully read over all of the new code you just wrote and other
existing code you just modified with "fresh eyes" looking super carefully for any obvious
bugs, errors, problems, issues, confusion, etc. Carefully fix anything you uncover.
Use ultrathink.
```
Run until no bugs found. 1–2 rounds for simple beads, 2–3 for complex. If 3+ rounds still
finding bugs, the implementation approach may be wrong — consider a different agent taking over.

### Advance to Next Bead
```
Reread AGENTS.md so it's still fresh in your mind. Use ultrathink. Use bv with the robot
flags to find the most impactful bead(s) to work on next and then start on it. Remember to
mark the beads appropriately and communicate with your fellow agents. Pick the next bead you
can actually do usefully now and start coding on it immediately; communicate what you're
working on to your fellow agents and mark beads appropriately as you work. And respond to
any agent mail messages you've received.
```

### Cross-Agent Review (alternate these two; run until both come back clean)

**Random Code Exploration:**
```
I want you to sort of randomly explore the code files in this project, choosing code files
to deeply investigate and understand and trace their functionality and execution flows
through the related code files which they import or which they are imported by. Once you
understand the purpose of the code in the larger context of the workflows, I want you to
do a super careful, methodical, and critical check with "fresh eyes" to find any obvious
bugs, problems, errors, issues, silly mistakes, etc. and then systematically and
meticulously correct them. Be sure to comply with ALL rules in AGENTS.md. Use ultrathink.
```

**Cross-Agent Review:**
```
Ok can you now turn your attention to reviewing the code written by your fellow agents
and checking for any issues, bugs, errors, problems, inefficiencies, security problems,
reliability issues, etc. and carefully diagnose their underlying root causes using
first-principle analysis and then fix or revise them if necessary? Don't restrict
yourself to the latest commits, cast a wider net and go super deep! Use ultrathink.
```

Send random exploration to 2–3 agents simultaneously (each explores different parts).
Alternate with cross-agent review. Two consecutive clean rounds = codebase is in good shape.

### Rust Quality Gates
```bash
cargo check --all-targets
cargo clippy --all-targets -- -D warnings
cargo fmt --check
```

---

## Step 9: Swarm Diagnosis

### Stuck Swarm Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Multiple agents pick same bead | Starts not staggered; not marking in_progress | Stagger, force Agent Mail claim messages |
| Agent circles after compaction | Forgot AGENTS.md operating contract | Force "Reread AGENTS.md"; restart if erratic |
| Bead stuck in_progress too long | Agent crashed or silently blocked | Check Agent Mail, reclaim bead, split blocker |
| Contradictory implementations | Not using Agent Mail + reservations | Audit reservation use, revise bead boundaries |
| Lots of commits, goal still far | Strategic drift | Stop, run reality check prompt, revise bead graph |

### Reality Check (when swarm looks busy but feels off)
```
Where are we on this project? Do we actually have the thing we are trying to build?
If not, what is blocking us? If we intelligently implement all open and in-progress
beads, would we close that gap completely? Why or why not?
```

If the answer is no — stop. Revise the bead graph. Re-aim the swarm.

---

## Step 10: Organized Commits

Every 1–2 hours, designate ONE agent for commits:

```
Now, based on your knowledge of the project, commit all changed files now in a series
of logically connected groupings with super detailed commit messages for each and then
push. Take your time to do it right. Don't edit the code at all. Don't commit obviously
ephemeral files. Use ultrathink.
```

One agent for commits prevents merge conflicts and produces coherent git history.
"Don't edit the code" is critical — without it, agents treat commits as "fix one more thing."

---

## Step 11: Landing the Session

Work is NOT complete until `git push` succeeds. Unpushed work is invisible to other agents.

1. File beads for any remaining work
2. Run quality gates (tests, linters, compiler)
3. Close finished beads, update in-progress
4. `br sync --flush-only` + `git add .beads/`
5. `git pull --rebase && git add <files> && git commit && git push`
6. `git status` must show "up to date with origin"

A session is only landable when the next swarm can pick it up from beads + AGENTS.md +
Agent Mail threads — without the human re-explaining the project from scratch.

---

## Agent Fungibility (Key Design Principle)

All agents are generalists. No specialist roles. No "ringleader" coordinator agent.
When any agent crashes:
1. The bead remains marked `in_progress`
2. Any other agent can resume it
3. Replace the dead agent with a fresh agent + marching orders prompt
4. No downtime, no data loss, minimal slowdown

Coordination lives in **artifacts** (beads, reservations, threads) and **tools** (bv, Agent Mail),
never in any special agent's state.

---

## De-Slopify Documentation (before shipping any docs)

```
I want you to read through the complete text carefully and look for any telltale signs
of "AI slop" style writing; one big tell is the use of emdash. You should try to replace
this with a semicolon, a comma, or just recast the sentence accordingly so it sounds good
while avoiding emdash. Also avoid: "It's not [just] XYZ, it's ABC", "Here's why",
"Let's dive in", "At its core...", "It's worth noting...". You MUST manually read each
line of the text and revise it manually in a systematic, methodical, diligent way.
```

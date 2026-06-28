# goal-sub

**A drop-in upgrade for `/goal` that keeps your parent context window clean by delegating all execution to a managed subagent network — so tasks of any length run reliably, without goal drift.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-skill-8A2BE2)](https://claude.ai/code)
[![Python 3.8+](https://img.shields.io/badge/Python-3.8%2B-yellow.svg)](https://www.python.org/)

---

## The Problem with Long Tasks in Claude Code

You give Claude a goal. It starts strong. Around tool call 40, something shifts — the original objective is still technically in context, but it's buried under layers of intermediate work, partial results, and cascading decisions. Claude starts optimizing for what's achievable *now* rather than what you asked for at the start.

This is goal drift. It's not a bug. It's a structural consequence of holding everything in one context window.

The `/goal` skill was built to fight this: it persists your objective to disk and injects it back on every stop. But it has one fundamental limit — **the parent context still does all the work.** Every file read, every edit, every decision accumulates in the same window that started the task. Eventually, that window fills.

`/goal-sub` removes that limit entirely.

---

## What `/goal-sub` Does Differently

| | `/goal` | `/goal-sub` |
|---|---|---|
| **Goal persistence** | SQLite, survives context resets | Same — reuses `/goal`'s SQLite |
| **Who does the work** | Parent Claude context, directly | Dedicated worker subagent network |
| **Context window pressure** | Accumulates with task length | Parent stays clean — always |
| **Parallel execution** | Not built in | Independent steps run simultaneously |
| **Task complexity ceiling** | Bounded by context window | Effectively unlimited |
| **Verification** | Completion audit before marking done | Dedicated verification subagent, file-evidence required |
| **State between steps** | In context | Written to `goal-state.json` after every step |

With `/goal`, Claude works like a developer who does everything themselves. Their desk gets cluttered, and eventually they lose track of what they started.

With `/goal-sub`, Claude works like a technical lead running a team. The lead stays focused on routing, decomposition, and verification. Each team member handles exactly one scoped task, produces a concrete artifact, and is done. The lead's context stays clean no matter how many tasks run.

---

## Architecture

The parent Claude **is** the orchestrator — there is no orchestrator subagent. The parent decomposes the goal, plans steps, spawns workers, and reads verification results. Only two categories of subagent ever exist: **workers** (one per step) and the **verifier** (one at the end).

```
/goal-sub <objective>
       │
       ├─ goal helper registers goal in SQLite
       ├─ workspace initialized at ~/.claude/goal-sub/workspaces/{session_id}/
       ├─ goal-state.json written to disk
       │
       └─ Parent Claude orchestrates directly
               │
               ├─ reads goal-state.json
               ├─ decomposes objective → writes step plan to goal-state.json
               │
               ├─ Round 0: spawn independent workers in parallel ──────┐
               │     Worker A  ──→  steps/step-01/output.md            │
               │     Worker B  ──→  steps/step-02/output.md            │
               │     Worker C  ──→  steps/step-03/output.md            │
               │                                                        │
               ├─ verify output files exist via Read/Glob ◄───────────┘
               ├─ update goal-state.json (steps_completed)
               │
               ├─ Round 1: spawn dependent workers ──→ ...
               │
               └─ Verification subagent (isolated context)
                       │
                       ├─ reads all step output files
                       └─ writes verification/report.md
```

**Key design rules:**

- **State is always on disk first.** Before any subagent is spawned, the relevant state is written to `goal-state.json`. Subagents read from disk, not from parent context.
- **File existence is the only valid completion signal.** The parent never trusts a subagent's return message — it verifies the output file exists using Read or Glob.
- **Parallel by default.** Steps with no dependency relationship are always spawned simultaneously in a single message.
- **Narrow scope per worker.** Each worker does exactly one step. This bounds context pressure and makes failures localized and recoverable.
- **Goal anchor in every subagent.** Every worker and the verifier receive the objective in a required `GOAL ANCHOR` block before any instructions. The goal is never assumed — always injected.

---

## Installation

**Prerequisites:** You must have the `/goal` skill installed first. `goal-sub` reuses its Python helper and SQLite state.

### Install both skills

**macOS / Linux:**
```bash
git clone https://github.com/risedial/goal-sub.git
cd goal-sub

cp -r .claude/skills/goal ~/.claude/skills/
cp -r .claude/skills/goal-sub ~/.claude/skills/
```

**Windows (PowerShell):**
```powershell
git clone https://github.com/risedial/goal-sub.git
cd goal-sub

Copy-Item -Recurse .claude\skills\goal "$env:USERPROFILE\.claude\skills\"
Copy-Item -Recurse .claude\skills\goal-sub "$env:USERPROFILE\.claude\skills\"
```

### Verify installation

Open Claude Code in any project and run:
```
/goal status
```

Expected output: `No goal is currently set for this Claude session.`

If Claude Code doesn't recognize the skill, restart the IDE or reload the window.

---

## Usage

```
/goal-sub Build a REST API with JWT auth, rate limiting, Postgres integration, and full test coverage
```

That's it. Claude will decompose this into steps, run them in parallel where possible, verify each output file, and report when done.

### All commands

```
/goal-sub <objective>              Set goal and begin subagent execution
/goal-sub --tokens 200K <obj>      Same with a soft token budget hint to workers
/goal-sub status                   Show current goal state (no subagent)
/goal-sub pause                    Pause execution
/goal-sub resume                   Resume a paused or interrupted goal
/goal-sub clear                    Delete the current goal
/goal-sub complete                 Mark complete after your own manual audit
```

`status`, `pause`, `resume`, `clear`, and `complete` run immediately in the parent context — no subagent needed.

---

## Execution Flow

When you run `/goal-sub <objective>`, the parent Claude executes these steps directly:

| Step | What happens |
|------|-------------|
| **1. Invoke helper** | `claude_goal.py invoke` registers the goal in SQLite and returns the action type |
| **2. Branch on action** | `set` → proceed to Step 3. `resume` → re-enter at correct step. `status/pause/clear/complete` → report and stop |
| **3. Initialize workspace** | Create `~/.claude/goal-sub/workspaces/{session_id}/steps/` and `verification/`. Write `goal-state.json` |
| **4. Decompose goal** | Parent analyzes the objective. Produces a step plan (1–8 steps) with dependencies. Writes plan to `goal-state.json` |
| **5. Execute rounds** | Steps with no unmet dependencies run in parallel. After each round, parent verifies output files exist and updates `goal-state.json` |
| **6. Spawn verifier** | One verification subagent reads all step outputs, maps them to the objective, writes `verification/report.md` |
| **7. Finalize** | Parent reads the report, updates `goal-state.json`, runs `claude_goal.py complete` if VERIFIED, reports to user |

---

## Resuming an Interrupted Goal

If a goal is interrupted mid-execution (Claude Code closed, session reset, manual pause), run:

```
/goal-sub resume
```

The parent reads `goal-state.json` to determine the exact re-entry point:

- **Steps incomplete** → re-enters at Step 5, skipping already-completed steps
- **All steps done, no verification report** → re-enters at Step 6 to spawn the verifier
- **Verification report exists** → re-enters at Step 7 to finalize and report

No work is repeated. State written to disk is the source of truth.

---

## The Stop Hook

Both `/goal` and `/goal-sub` register a Claude Code `Stop` hook. When Claude would normally stop between turns, the hook fires and injects the current objective back into context, keeping Claude working automatically.

Set a goal, walk away, come back to a completed task — without pressing Enter again.

The hook stops firing when you run `/goal-sub pause`, `/goal-sub clear`, or `/goal-sub complete`. A runaway guard (default: 500 continuations, configurable via `CLAUDE_GOAL_MAX_STOP_CONTINUES`) prevents infinite loops.

---

## State Files

All session state is stored in platform-native locations:

| File | Purpose |
|------|---------|
| `~/.claude/goal/goals.sqlite` | Goal registration, session tracking, status, elapsed time |
| `~/.claude/goal-sub/workspaces/{session_id}/goal-state.json` | Orchestrator state, step plan, completion record |
| `~/.claude/goal-sub/workspaces/{session_id}/steps/{id}/output.md` | Worker deliverable |
| `~/.claude/goal-sub/workspaces/{session_id}/steps/{id}/status.json` | Worker completion signal |
| `~/.claude/goal-sub/workspaces/{session_id}/verification/report.md` | Final verification verdict |

State persists between Claude Code sessions. If a task is interrupted, workspace files remain and can be inspected directly.

---

## goal-state.json Schema

The `goal-state.json` file is the source of truth for all orchestration state. It is written before any subagent is spawned and updated after every round.

```json
{
  "schema_version": "1.0",
  "session_id": "term:a3f1b2c4d5e6f7a8",
  "objective": "Build a REST API with JWT auth and full test coverage",
  "workspace": "/Users/alex/.claude/goal-sub/workspaces/term:a3f1b2c4d5e6f7a8",
  "status": "executing",
  "orchestrator": {
    "status": "executing",
    "step_plan": [
      {
        "id": "step-01",
        "name": "Scaffold API structure",
        "description": "Create project layout, package.json, and base Express setup",
        "dependencies": [],
        "output_file": "...workspaces/{id}/steps/step-01/output.md",
        "status_file": "...workspaces/{id}/steps/step-01/status.json"
      }
    ],
    "steps_completed": ["step-01"],
    "steps_total": 4
  },
  "completion": {
    "status": "PENDING",
    "confidence": null,
    "evidence": [],
    "report_file": null
  },
  "created_at": "2025-06-27T14:00:00Z",
  "updated_at": "2025-06-27T14:12:33Z"
}
```

**`orchestrator.status`** values: `pending` → `executing` → `complete`

**`completion.status`** values (set at Step 7 from the verification report):
- `VERIFIED` — all deliverables confirmed by tool-call evidence
- `PARTIAL` — some deliverables complete, gaps remain
- `NOT_ACHIEVED` — objective not achieved; user is asked whether to continue

---

## Why Subagents Beat a Single Long Context

Claude's context window is a shared resource. Every file read, every tool call result, every intermediate decision competes for the same space. Long tasks don't just consume tokens — they dilute focus. The most recent content crowds out the original objective.

Subagents solve this structurally, not behaviorally:

- **Each worker starts fresh.** No accumulated context. No goal drift. Scope is bounded to one step.
- **The goal anchor is injected, not remembered.** Every subagent receives the objective in a required header before any instructions. It reads from disk — never assumes.
- **Parallel execution is the default.** Steps with no dependency relationship run simultaneously, not sequentially.
- **File existence is the only valid completion signal.** The orchestrator doesn't trust what a subagent says it did. It checks whether the output file exists.

This is the same architectural principle behind production distributed systems: don't hold critical state in memory. Write it to disk. Let each worker read what it needs and nothing more.

---

## Requirements

- [Claude Code](https://claude.ai/code) (any version with skill support)
- Python 3.8+ (for the goal helper script — zero external dependencies, pure stdlib)
- The `/goal` skill (included in this repo)

---

## Contributing

Bug reports and pull requests are welcome. For significant changes, open an issue first to discuss the approach. When submitting a PR:

1. Ensure changes to `claude_goal.py` include corresponding test coverage
2. Update relevant sections of this README if the behavior changes
3. Keep the SKILL.md accurate — it is the contract between the helper script and Claude's orchestration logic

---

## License

MIT

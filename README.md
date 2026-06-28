# goal-sub

**A drop-in upgrade for `/goal` that delegates all execution to a self-directing subagent network — so your context window never bloats, no matter how long or complex the task.**

---

## The Problem with Long Tasks in Claude Code

You give Claude a goal. It starts strong. Then, around tool call 40, something shifts. The original objective is still technically in context — but it's buried under layers of intermediate work, partial results, and cascading decisions. Claude starts optimizing for what's achievable *now* rather than what you asked for at the start.

This is goal drift. It's not a bug in Claude. It's a structural consequence of holding everything in one context window.

The `/goal` skill was built to fight this. It persists your objective to disk and injects it back on every stop, keeping Claude oriented across long sessions. But it has one fundamental limit: **the parent context still does all the work.** Every file read, every edit, every decision accumulates in the same window that started the task. Eventually, that window fills.

`/goal-sub` removes that limit entirely.

---

## What `/goal-sub` Does Differently

| | `/goal` | `/goal-sub` |
|---|---|---|
| **Goal persistence** | SQLite, survives context resets | Same — reuses `/goal`'s SQLite |
| **Who does the work** | Parent Claude context, directly | Dedicated subagent network |
| **Context window pressure** | Accumulates with task length | Parent stays clean — always |
| **Parallel execution** | Not built in | Independent steps run simultaneously |
| **Task complexity ceiling** | Bounded by context window | Effectively unlimited |
| **Verification** | Completion audit before marking done | Dedicated verification subagent with file-evidence requirement |
| **State between steps** | In context | Written to `goal-state.json` after every step |

With `/goal`, Claude works like a single developer who does everything themselves. Their desk gets cluttered, and eventually they lose track of what they started working on.

With `/goal-sub`, Claude works like a technical lead running a team. The lead stays focused on routing and verification. Each team member handles one scoped task, produces a concrete artifact, and is done. The lead's desk stays clean no matter how many tasks are in flight.

---

## How It Works

When you run `/goal-sub <your objective>`:

1. **Goal is registered** via the same SQLite helper as `/goal` — same session tracking, same stop hook
2. **Workspace is initialized** at `~/.claude/goal-sub/workspaces/{session_id}/`
3. **Orchestrator subagent is spawned** — it receives the objective and a blank workspace; it never inherits the parent's context
4. **Orchestrator decomposes** the objective into 1–8 concrete steps with a dependency graph
5. **Workers are spawned in parallel rounds** — all steps with no unmet dependencies run simultaneously
6. **Each worker** reads the goal anchor, does exactly one step, writes an output file, writes a status JSON
7. **Orchestrator verifies** each output file exists before marking the step complete
8. **Verification subagent** reads all outputs and maps them to the original objective
9. **Parent reads the verification report** and marks the goal complete via the goal helper

The parent Claude context does none of the domain work. It spawns, waits, verifies a file exists, and reports. That's it.

---

## Architecture

```
/goal-sub <objective>
       │
       ├─ goal helper registers goal in SQLite
       ├─ workspace initialized: ~/.claude/goal-sub/workspaces/{session_id}/
       ├─ goal-state.json written to disk
       │
       └─ Orchestrator subagent (isolated context)
               │
               ├─ reads goal-state.json
               ├─ writes step plan to goal-state.json
               │
               ├─ Round 0: spawn independent workers in parallel ──┐
               │     Worker A  ──→  output.md + status.json        │
               │     Worker B  ──→  output.md + status.json        │
               │     Worker C  ──→  output.md + status.json        │
               │                                                    │
               ├─ verify output files exist ◄────────────────────┘
               ├─ update goal-state.json
               │
               ├─ Round 1: spawn dependent workers ──→ ...
               │
               └─ Verification subagent
                       │
                       ├─ reads all step output files
                       └─ writes verification/report.md
```

Every piece of reasoning that needs to survive a context boundary is written to disk first. No state lives only in memory.

---

## Installation

**Prerequisites:** You must have the `/goal` skill installed first. `goal-sub` reuses its Python helper and SQLite state.

### Install both skills

Copy the `.claude` folder contents into your global Claude Code config:

**macOS / Linux:**
```bash
# Clone the repo
git clone https://github.com/risedial/goal-sub.git
cd goal-sub

# Install both skills
cp -r .claude/skills/goal ~/.claude/skills/
cp -r .claude/skills/goal-sub ~/.claude/skills/
```

**Windows (PowerShell):**
```powershell
# Clone the repo
git clone https://github.com/risedial/goal-sub.git
cd goal-sub

# Install both skills
Copy-Item -Recurse .claude\skills\goal "$env:USERPROFILE\.claude\skills\"
Copy-Item -Recurse .claude\skills\goal-sub "$env:USERPROFILE\.claude\skills\"
```

### Verify installation

Open Claude Code in any project and run:
```
/goal status
```

You should see: `No goal is currently set for this Claude session.`

If Claude Code doesn't recognize the skill, restart the IDE or reload the window.

---

## Usage

```
/goal-sub Build a REST API with JWT auth, rate limiting, Postgres integration, and full test coverage
```

That's it. Claude will decompose this into steps, run them in parallel where possible, verify each output, and report when done.

### All commands

```
/goal-sub <objective>              Set goal and begin subagent execution
/goal-sub --tokens 200K <obj>      Same with a soft token budget
/goal-sub status                   Check current goal state
/goal-sub pause                    Pause execution (stop hook will no longer fire)
/goal-sub resume                   Resume a paused goal
/goal-sub clear                    Delete the current goal
/goal-sub complete                 Mark complete after your own manual audit
```

Status, pause, resume, clear, and complete run immediately in the parent context — no subagent needed.

---

## What the Stop Hook Does

Both `/goal` and `/goal-sub` register a Claude Code Stop hook. When Claude would normally stop between turns, the hook fires and injects the current objective back into context, keeping Claude working toward the goal automatically.

This means you can set a goal, walk away, and come back to a completed task — without needing to press Enter again.

The hook stops firing when you run `/goal-sub pause`, `/goal-sub clear`, or `/goal-sub complete`. There is also a runaway guard (default: 500 continuations) to prevent infinite loops.

---

## State Files

All session state is stored in platform-native locations:

| File | Purpose |
|------|---------|
| `~/.claude/goal/goals.sqlite` | Goal registration, session tracking, status |
| `~/.claude/goal-sub/workspaces/{session_id}/goal-state.json` | Orchestrator state, step plan, completion record |
| `~/.claude/goal-sub/workspaces/{session_id}/steps/{id}/output.md` | Worker deliverable |
| `~/.claude/goal-sub/workspaces/{session_id}/steps/{id}/status.json` | Worker completion signal |
| `~/.claude/goal-sub/workspaces/{session_id}/verification/report.md` | Final verification |

State persists between Claude Code sessions. If a task is interrupted, the workspace files remain and can be inspected.

---

## Why Subagents Beat a Single Long Context

Claude's context window is a shared resource. Every file read, every tool call result, every intermediate decision competes for the same space. Long tasks don't just consume tokens — they dilute focus. The most recent content crowds out the original objective.

Subagents solve this structurally, not behaviorally:

- **Each subagent starts fresh.** No accumulated context. No goal drift.
- **Scope is narrow.** One step. One output file. One status signal. Context pressure is bounded.
- **The goal anchor is injected structurally.** Every subagent receives the objective in a required header it must read before acting. The goal isn't remembered — it's read from disk.
- **File existence is the only valid completion signal.** The orchestrator doesn't trust what a subagent says it did. It checks whether the output file exists.

This is the same architectural principle behind production distributed systems: don't hold critical state in memory. Write it to disk. Let each worker read what it needs and nothing more.

---

## Requirements

- Claude Code (any version with skill support)
- Python 3.8+ (for the goal helper script)
- The `/goal` skill (included in this repo)

No additional dependencies. The goal helper script is pure Python stdlib.

---

## License

MIT

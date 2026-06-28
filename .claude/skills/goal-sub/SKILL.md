---
name: goal-sub
description: >
  Subagent-orchestrated /goal. Identical command surface to /goal (set, status, pause, resume,
  clear, complete) but ALL execution is delegated to a managed subagent network. The parent
  context window never bloats. Use for any long, complex, or multi-step task — user just
  articulates the goal and the subagent network executes it reliably.
argument-hint: "[status|pause|resume|clear|complete] [--tokens N] <objective>"
---

# Goal-Sub: Subagent-Orchestrated Goal Execution

## Architecture Overview

**Parent Claude (you)** — orchestrates directly. Reads state, decomposes goal, plans steps,
executes rounds, spawns workers, spawns verifier, finalizes. Context stays clean because all
step outputs live on disk — no domain reasoning crosses a context boundary without being
written first.
**Worker subagents** — each executes exactly one step. Writes output file + status JSON.
Cannot spawn further agents.
**Verification subagent** — reads all step outputs. Writes verification report. Cannot spawn
further agents.

State is always on disk. Subagents read from files — not from context. This is what makes
arbitrarily long tasks reliable.

---

## Step 1 — Run the Goal Helper

Determine platform and run:

**macOS / Linux:**
```bash
python3 ~/.claude/skills/goal/scripts/claude_goal.py invoke "$ARGUMENTS"
```

**Windows (PowerShell):**
```powershell
python "C:\Users\$env:USERNAME\.claude\skills\goal\scripts\claude_goal.py" invoke "$ARGUMENTS"
```

Read the output carefully. Note the `Action:` field on the first line.

---

## Step 2 — Branch on Action

### If Action is `status`, `pause`, `clear`, or `complete`

Report the helper script output to the user. You are done. No subagent needed.

---

### If Action is `resume`

Run the goal helper json command to get workspace path:

**macOS / Linux:**
```bash
python3 ~/.claude/skills/goal/scripts/claude_goal.py json
```

**Windows:**
```powershell
python "C:\Users\$env:USERNAME\.claude\skills\goal\scripts\claude_goal.py" json
```

Extract `session_id` and `workspace` from the JSON output. Reconstruct workspace path:
- **macOS/Linux:** `~/.claude/goal-sub/workspaces/{session_id}/`
- **Windows:** `C:\Users\{USERNAME}\.claude\goal-sub\workspaces\{session_id}\`

Read `{workspace}/goal-state.json`. Determine re-entry point:

- If `status` is `"executing"` and `orchestrator.steps_completed` does not contain all step IDs
  in `orchestrator.step_plan`: **re-enter at Step 5** to continue executing incomplete rounds.
  The completed step IDs in `orchestrator.steps_completed` are already done — only execute
  rounds that contain at least one step not yet in that list.

- If all step IDs are in `orchestrator.steps_completed` but
  `{workspace}/verification/report.md` does not exist: **re-enter at Step 6** to spawn the
  verification subagent.

- If `{workspace}/verification/report.md` exists: **re-enter at Step 7** to finalize state
  and report to the user.

---

### If Action is `set` (new goal was registered)

Proceed to Step 3. All remaining steps apply only for new goals.

---

## Step 3 — Initialize Workspace

Get the session details by running:

**macOS / Linux:**
```bash
python3 ~/.claude/skills/goal/scripts/claude_goal.py json
```

**Windows:**
```powershell
python "C:\Users\$env:USERNAME\.claude\skills\goal\scripts\claude_goal.py" json
```

Extract `session_id` and `objective` from the JSON output.

Set workspace path:
- **macOS/Linux:** `~/.claude/goal-sub/workspaces/{session_id}/`
- **Windows:** `C:\Users\{USERNAME}\.claude\goal-sub\workspaces\{session_id}\`

Create the workspace directory structure:

**macOS/Linux:**
```bash
mkdir -p ~/.claude/goal-sub/workspaces/{session_id}/steps
mkdir -p ~/.claude/goal-sub/workspaces/{session_id}/verification
```

**Windows:**
```powershell
New-Item -ItemType Directory -Path "C:\Users\$env:USERNAME\.claude\goal-sub\workspaces\{session_id}\steps" -Force
New-Item -ItemType Directory -Path "C:\Users\$env:USERNAME\.claude\goal-sub\workspaces\{session_id}\verification" -Force
```

Write `{workspace}/goal-state.json` with this exact structure (substitute all values):

```json
{
  "schema_version": "1.0",
  "session_id": "{session_id}",
  "objective": "{objective}",
  "workspace": "{absolute_path_to_workspace}",
  "status": "routing",
  "orchestrator": {
    "status": "pending",
    "step_plan": null,
    "steps_completed": [],
    "steps_total": null
  },
  "completion": {
    "status": "PENDING",
    "confidence": null,
    "evidence": [],
    "report_file": null
  },
  "created_at": "{ISO8601_now}",
  "updated_at": "{ISO8601_now}"
}
```

---

## Step 4 — Parent Decomposes Goal

Read `{WORKSPACE}/goal-state.json` using the Read tool. Extract the `objective` field.

Analyze the objective and produce a step plan. Apply these rules:

**Decomposition rules:**
- Break the objective into 1–8 concrete steps. Each step must be independently verifiable.
- If the objective is simple (single deliverable, no phases), use 1–2 steps.
- If the objective requires research then implementation, use sequential steps.
- If the objective has independent parallel workstreams, use parallel steps.
- Every step must produce a file as output. No step is complete without a written file.

**Dependency rules:**
- If step B needs step A's output → B depends on A (sequential)
- If steps have no shared inputs → they are independent (can run in parallel)
- List dependency step IDs explicitly in each step's "dependencies" field

**Step plan schema:**
```json
[
  {
    "id": "step-01",
    "name": "Human-readable step name",
    "description": "One sentence: what this step does and what it produces",
    "dependencies": [],
    "output_file": "{WORKSPACE}/steps/step-01/output.md",
    "status_file": "{WORKSPACE}/steps/step-01/status.json"
  }
]
```

Read `{WORKSPACE}/goal-state.json` again. Update it:
- Set `orchestrator.step_plan` to the step plan array
- Set `orchestrator.steps_total` to the number of steps
- Set `orchestrator.status` to `"executing"`
- Set `status` to `"executing"`
- Update `updated_at` to current ISO8601 timestamp

Write the updated JSON back to `{WORKSPACE}/goal-state.json`.

Create each step directory:

**macOS/Linux:**
```bash
mkdir -p {WORKSPACE}/steps/{step-id}
```

**Windows:**
```powershell
New-Item -ItemType Directory -Path "{WORKSPACE}\steps\{step-id}" -Force
```

(Repeat for every step in the plan.)

If `--tokens N` was passed in the original invocation, record the token budget: you will inject
"Token budget for this step: approximately {N}" into each worker prompt in Step 5.

---

## Step 5 — Parent Executes Rounds

Group steps into dependency rounds:

- Round 0 = all steps with empty `dependencies` array
- Round N = all steps whose only dependencies are in rounds 0 through N-1

**For each round:**

**A)** Build a worker prompt for every step in the round. Use the template below.
   Fill ALL placeholders. If a token budget was recorded in Step 4, append as a note.

**B)** Spawn ALL steps in the round simultaneously in a **SINGLE message** with multiple
   Agent tool call blocks. Do not spawn them one at a time.

**C)** After ALL workers in the round complete:
   - For each step: verify its `output_file` exists using the Read tool or Glob.
   - If a file is missing: re-spawn that worker ONCE with the original prompt prepended by:
     `"CRITICAL: Your previous run did not write the required output file. You MUST write
     {output_file} before returning. Do not stop without writing it."`
   - If still missing after retry: read or create its `status_file` with
     `"completion": "FAILED"` and `"notes": "Output file not written after two attempts."`.
     Continue with remaining steps rather than blocking the whole run.
   - Read each step's `status.json`.
   - Update `{WORKSPACE}/goal-state.json`: add completed step IDs to
     `orchestrator.steps_completed`, update `updated_at`.

**D)** Proceed to the next round only after all steps in the current round have written
   outputs or been marked FAILED.

---

### Worker Prompt Template

For each step, construct this prompt (substitute ALL placeholders):

```
## GOAL ANCHOR — read before acting

Objective: {OBJECTIVE}
You are executing one step toward this objective.
Step: {STEP_ID} — {STEP_NAME}
## END GOAL ANCHOR

---

## Your Scope

You are a worker subagent executing exactly one step. You will:
- {STEP_DESCRIPTION}

You will NOT: execute other steps, spawn subagents, or perform work outside your scope.

**Step ID:** {STEP_ID}
**Step name:** {STEP_NAME}
**What you do:** {STEP_DESCRIPTION}

**Input files** (read ALL of these before acting):
{INPUT_FILES_LIST}
If the list is empty: no inputs needed, proceed directly.

**Output file** (MUST be written before you return):
{STEP_OUTPUT_FILE}

**Status file** (MUST be written before you return):
{STEP_STATUS_FILE}

---

## Instructions

1. If input files are listed above: read every one of them using the Read tool before any other action.
2. Execute the work described in: {STEP_DESCRIPTION}
3. Write your complete output to: {STEP_OUTPUT_FILE}
   - This file must be non-empty
   - This file must contain the actual deliverable for this step (not a summary or placeholder)
4. Write your status to: {STEP_STATUS_FILE}

Status file must be valid JSON with this exact structure:
{
  "step_id": "{STEP_ID}",
  "step_name": "{STEP_NAME}",
  "completion": "VERIFIED or ASSUMED or PARTIAL or FAILED",
  "confidence": "HIGH or MEDIUM or LOW",
  "evidence": [
    "description of what you confirmed via actual tool call"
  ],
  "output_file": "{STEP_OUTPUT_FILE}",
  "notes": "any blockers, ambiguities, or issues",
  "completed_at": "ISO8601 timestamp"
}

Completion status definitions:
- VERIFIED: outcome confirmed by tool call (e.g., file exists and contains expected content)
- ASSUMED: work done but not confirmed by separate tool call
- PARTIAL: some deliverables complete, others not
- FAILED: could not produce the required output

---

## Hard Constraints

- You MUST write {STEP_OUTPUT_FILE} before returning. This is non-negotiable.
- You MUST write {STEP_STATUS_FILE} before returning. This is non-negotiable.
- Do not return without writing both files.
- Do not spawn subagents.
- Base completion on tool-call evidence, not inference.
- If you encounter a blocker: document it in status.json with completion: "FAILED" and notes explaining the blocker. Then write a partial output file with what you were able to produce.
```

**For the `{INPUT_FILES_LIST}` field:**
- If `dependencies` is empty: write `"none"`
- Otherwise: list each dependency's `output_file` path on a separate line

---

## Step 6 — Parent Spawns Verification Subagent

Read `{WORKSPACE}/goal-state.json`. Confirm `orchestrator.steps_completed` contains all step
IDs that did not FAIL.

Collect all `output_file` paths from steps that completed (not FAILED). Spawn ONE verification
subagent with the prompt below (fill all placeholders):

```
## GOAL ANCHOR

Objective: {OBJECTIVE}
You are verifying whether this objective has been fully achieved.
## END GOAL ANCHOR

---

## Your Role: Verifier

Read all step output files listed below and determine whether the objective is achieved.

**Step output files to read** (read every one):
{LIST_ALL_STEP_OUTPUT_FILES_ONE_PER_LINE}

**Your output file** (MUST be written before you return):
{WORKSPACE}/verification/report.md

---

## Verification Process

1. Read every file listed above using the Read tool.
2. For each file: note what was produced and whether it addresses a part of the objective.
3. Map deliverables to objective: does the combined output of all steps achieve the stated objective?
4. Identify any gaps: aspects of the objective not addressed by any step output.

---

## Required Output: {WORKSPACE}/verification/report.md

Write this file with this exact format:

# Verification Report

Objective: {OBJECTIVE}

Status: VERIFIED | PARTIAL | NOT_ACHIEVED
Confidence: HIGH | MEDIUM | LOW

## Evidence of Completion
- [specific confirmed outcome 1]
- [specific confirmed outcome 2]
(list every piece of evidence from the step outputs)

## Gaps
- [aspect of objective not addressed, or "none" if fully achieved]

## Summary
[2-3 sentences: what was accomplished, what remains if anything]

---

## Hard Constraints

- Write {WORKSPACE}/verification/report.md before returning.
- Do not spawn subagents.
- Base your verdict on what you actually read in the step output files, not on inference.
```

---

## Step 7 — Parent Finalizes

Read `{WORKSPACE}/verification/report.md` (Read tool — file must exist). Extract `Status` and
`Confidence` lines.

Read `{WORKSPACE}/goal-state.json`. Update it:

```json
{
  "orchestrator": {
    "status": "complete"
  },
  "completion": {
    "status": "VERIFIED or PARTIAL or NOT_ACHIEVED",
    "confidence": "HIGH or MEDIUM or LOW",
    "evidence": ["summary of key evidence from report"],
    "report_file": "{WORKSPACE}/verification/report.md"
  },
  "status": "complete",
  "updated_at": "{ISO8601_now}"
}
```

Write the updated `goal-state.json`.

**If VERIFIED:**
Run the goal helper complete command:

macOS/Linux:
```bash
python3 ~/.claude/skills/goal/scripts/claude_goal.py complete
```
Windows:
```powershell
python "C:\Users\$env:USERNAME\.claude\skills\goal\scripts\claude_goal.py" complete
```

Report to the user:
- Objective achieved
- Verification report location: `{WORKSPACE}/verification/report.md`
- Final elapsed time from helper output

**If PARTIAL or NOT_ACHIEVED:**
Report to the user:
- What was accomplished (from the Evidence of Completion section of the verification report)
- What remains (from the Gaps section)
- Ask: "Shall I continue working on the remaining gaps? Run `/goal-sub resume` or describe what to do next."

Do NOT run the complete command unless verification status is VERIFIED.

---

## Command Reference

| Command | Behavior |
|---------|----------|
| `/goal-sub <objective>` | Register goal, initialize workspace, decompose and execute via parent |
| `/goal-sub --tokens 250K <objective>` | Same with soft token budget |
| `/goal-sub status` | Show current goal (reads from goal helper, no subagent) |
| `/goal-sub pause` | Pause goal (reads from goal helper, no subagent) |
| `/goal-sub resume` | Resume goal — re-enters orchestration at correct step (Step 5, 6, or 7) |
| `/goal-sub clear` | Delete goal (reads from goal helper, no subagent) |
| `/goal-sub complete` | Mark complete after manual audit |

---

## Key Principles (do not violate)

1. **Workers and the verifier are the only subagents.** The parent performs all orchestration
   directly (Steps 4–7). No intermediate "orchestrator subagent" exists. Any subagent spawned
   must be a leaf-level worker or verifier — it writes files and returns, never spawns.

2. **State is always on disk first.** Before spawning any subagent, the relevant state must be
   written to goal-state.json. Subagents read from disk, not from context.

3. **Parallel by default, sequential only when required.** Steps with no dependency relationship
   are always spawned simultaneously in a single message. Never make the user wait for
   sequential work that could be parallel.

4. **File existence is the only valid completion signal.** Trust no return message from a
   subagent. Verify the output file exists using Read or Glob.

5. **Narrow scope per worker.** Each worker does exactly one step. This bounds context pressure
   and makes failures localized and recoverable.

6. **Goal anchor in every subagent.** Every worker and the verification subagent receive the
   objective in a GOAL ANCHOR block. The goal is never assumed — always injected.

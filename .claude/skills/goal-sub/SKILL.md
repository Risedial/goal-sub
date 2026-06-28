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

**Parent Claude (you)** — routes only. Never performs domain work. Stays clean.
**Orchestrator subagent** — decomposes goal, plans steps, spawns workers, aggregates, verifies.
**Worker subagents** — each executes one step, writes output file + status JSON, never more.

State is always on disk. Subagents read from files — not from context. No reasoning crosses a
context boundary without being written first. This is what keeps the parent clean and makes
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
python "$env:USERPROFILE\.claude\skills\goal\scripts\claude_goal.py" invoke "$ARGUMENTS"
```

Read the output carefully. Note the `Action:` field on the first line.

---

## Step 2 — Branch on Action

### If Action is `status`, `pause`, `resume`, `clear`, or `complete`

Report the helper script output to the user. You are done. No subagent needed.

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
python "$env:USERPROFILE\.claude\skills\goal\scripts\claude_goal.py" json
```

Extract `session_id` and `objective` from the JSON output.

Determine the workspace base path:
- **macOS/Linux:** `~/.claude/goal-sub/workspaces/{session_id}/`
- **Windows:** resolve `$env:USERPROFILE\.claude\goal-sub\workspaces\{session_id}\`

Create the workspace directory structure:

**macOS/Linux:**
```bash
mkdir -p ~/.claude/goal-sub/workspaces/{session_id}/steps
mkdir -p ~/.claude/goal-sub/workspaces/{session_id}/verification
```

**Windows:**
```powershell
$base = "$env:USERPROFILE\.claude\goal-sub\workspaces\{session_id}"
New-Item -ItemType Directory -Path "$base\steps" -Force
New-Item -ItemType Directory -Path "$base\verification" -Force
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

## Step 4 — Spawn the Orchestrator Subagent

Spawn ONE orchestrator subagent using the Agent tool. Pass the constructed prompt below with
ALL placeholders filled. The parent waits for it to complete. Do not do any domain work yourself.

**Construct the orchestrator prompt by substituting:**
- `{OBJECTIVE}` → the full objective string from goal-state.json
- `{WORKSPACE}` → the absolute workspace path (no trailing slash)
- `{SESSION_ID}` → the session ID string

---

```
## GOAL ANCHOR — read this before any other action

Objective: {OBJECTIVE}
Session: {SESSION_ID}
Goal state file: {WORKSPACE}/goal-state.json

You MUST read goal-state.json before taking any action.
You MUST NOT report completion without file-existence evidence for every required output.
COMPLETION (VERIFIED / ASSUMED / PARTIAL / FAILED) and CONFIDENCE (HIGH / MEDIUM / LOW) are
always reported as separate fields — never combined.

## END GOAL ANCHOR

---

## Your Role: Orchestrator

You are the orchestrator for this goal-sub execution. Your job has exactly four responsibilities:

1. Decompose the objective into concrete, verifiable steps
2. Spawn worker subagents to execute steps (parallel where independent, sequential where dependent)
3. Verify each worker produced its required output file
4. Spawn a verification subagent and finalize the goal-state.json

You WILL NOT perform domain work directly. You WILL NOT write code, research topics, create
files with content, or execute task steps yourself. All of that goes to worker subagents.
Your only outputs are: spawning subagents, verifying file existence, and writing goal-state.json.

---

## Workspace

All files for this session live here: {WORKSPACE}
Goal state file: {WORKSPACE}/goal-state.json
Steps directory: {WORKSPACE}/steps/
Verification directory: {WORKSPACE}/verification/

---

## Phase 1: Read State and Create Step Plan

Read {WORKSPACE}/goal-state.json using the Read tool.
Extract the objective field.

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

Read {WORKSPACE}/goal-state.json again. Update it to add the step plan:
- Set `orchestrator.step_plan` to the step plan array
- Set `orchestrator.steps_total` to the number of steps
- Set `orchestrator.status` to "executing"
- Set `status` to "executing"
- Update `updated_at` to current ISO8601 timestamp

Write the updated JSON back to {WORKSPACE}/goal-state.json.

Create each step directory. For each step in the plan:
- Create {WORKSPACE}/steps/{step-id}/ directory

---

## Phase 2: Execute Steps in Dependency Order

Determine the execution order: group steps into rounds where all steps in a round have
their dependencies already completed.

Round 0 = all steps with empty dependencies array.
Round 1 = all steps whose only dependencies are in Round 0. And so on.

**For each round:**

A) Construct a worker prompt for every step in this round. Use the template below.
   Fill ALL placeholders before spawning.

B) Spawn ALL steps in this round simultaneously in a SINGLE Agent tool call message
   (multiple Agent blocks in one response). Do not spawn them one at a time.

C) After ALL workers in the round complete:
   - For each step: verify its output_file exists using the Read tool or Glob
   - If a file is missing: re-spawn that worker ONCE with the original prompt prepended by:
     "CRITICAL: Your previous run did not write the required output file. You MUST write
     {output_file} before returning. Do not stop without writing it."
   - Read each step's status.json
   - Update {WORKSPACE}/goal-state.json: add each step ID to `orchestrator.steps_completed`,
     update `updated_at`

D) Proceed to the next round.

---

### Worker Prompt Template

For each step, build this prompt (substitute ALL placeholders):

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
- Base completion on tool-call evidence, not inference.
- If you encounter a blocker: document it in status.json with completion: "FAILED" and notes explaining the blocker. Then write a partial output file with what you were able to produce.
```

**For the {INPUT_FILES_LIST} field:**
- If dependencies is empty: write "none"
- Otherwise: list each dependency's output_file path on a separate line

---

## Phase 3: Spawn Verification Subagent

After all steps complete, read {WORKSPACE}/goal-state.json to confirm
`orchestrator.steps_completed` contains all step IDs.

Spawn one verification subagent with this prompt (fill all placeholders):

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

## Required Output

Write {WORKSPACE}/verification/report.md with this exact format:

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

You MUST write this file before returning. Base your verdict on what you actually read
in the step output files, not on inference about what should have been produced.
```

---

## Phase 4: Finalize State and Return

After the verification subagent returns:

1. Verify {WORKSPACE}/verification/report.md exists (Read it)
2. Extract Status and Confidence from the report

Read {WORKSPACE}/goal-state.json. Update it:
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

Write the updated goal-state.json.

Return this summary to the parent:

```
ORCHESTRATION COMPLETE

Objective: {OBJECTIVE}
Completion: [Status from verification report]
Confidence: [Confidence from verification report]
Steps executed: [count]
Workspace: {WORKSPACE}

Key outputs:
[list each step ID and its output file path]

Verification report: {WORKSPACE}/verification/report.md

[If PARTIAL or NOT_ACHIEVED: list what gaps remain]
```
```

---

## Step 5 — After Orchestrator Returns

Read `{WORKSPACE}/goal-state.json` to confirm `completion.status`.

**If VERIFIED:**
Run the goal helper complete command:

macOS/Linux:
```bash
python3 ~/.claude/skills/goal/scripts/claude_goal.py complete
```
Windows:
```powershell
python "$env:USERPROFILE\.claude\skills\goal\scripts\claude_goal.py" complete
```

Report to the user:
- Objective achieved
- Verification report location
- Final elapsed time from helper output

**If PARTIAL or NOT_ACHIEVED:**
Report to the user:
- What was accomplished (from verification report gaps section)
- What remains
- Ask: "Shall I continue working on the remaining gaps? If so, run `/goal-sub resume` or describe what to do next."

Do NOT run the complete command unless verification status is VERIFIED.

---

## Command Reference

| Command | Behavior |
|---------|----------|
| `/goal-sub <objective>` | Register goal, initialize workspace, spawn orchestrator, execute |
| `/goal-sub --tokens 250K <objective>` | Same with soft token budget |
| `/goal-sub status` | Show current goal (reads from goal helper, no subagent) |
| `/goal-sub pause` | Pause goal (reads from goal helper, no subagent) |
| `/goal-sub resume` | Resume goal (reads from goal helper, no subagent) |
| `/goal-sub clear` | Delete goal (reads from goal helper, no subagent) |
| `/goal-sub complete` | Mark complete after manual audit |

---

## Key Principles (do not violate)

1. **Parent never does domain work.** If you find yourself writing code, researching, or creating content directly — stop and delegate to a subagent.

2. **State is always on disk first.** Before spawning any subagent, the relevant state must be written to goal-state.json. Subagents read from disk, not from context.

3. **Parallel by default, sequential only when required.** Steps with no dependency relationship are always spawned simultaneously. Never make the user wait for sequential work that could be parallel.

4. **File existence is the only valid completion signal.** Trust no return message from a subagent. Verify the output file exists.

5. **Narrow scope per worker.** Each worker does exactly one step. This bounds context pressure and makes failures localized and recoverable.

6. **Goal anchor in every subagent.** Every worker and the verification subagent receive the objective in a GOAL ANCHOR block. The goal is never assumed — always injected.

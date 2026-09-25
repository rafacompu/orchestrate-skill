---
name: orchestrate
description: "Turn the plan just discussed in this conversation into a Solo-orchestrated fan-out: scratchpad plan, todos with blockers, parallel worker agents, idle-wake supervision, and clean close-out. Invoke explicitly with /orchestrate after a solution has been discussed."
argument-hint: "(optional) extra steering notes"
disable-model-invocation: true
---

# /orchestrate — plan → Solo fan-out → close-out

The user typed `/orchestrate`. A solution has already been discussed in this
conversation. Turn that discussion into a supervised, parallel Solo workstream.
`$ARGUMENTS`, if present, is extra steering to fold into the plan.

Use the Solo MCP tools throughout (`mcp__solo__*`). If Solo project scope is
unset, call `list_projects` / `select_project` first.

## Core principles (do not violate)

- **Brief via scratchpad key.** Workers read their full context from the
  scratchpad. Spawn prompts stay tiny — never paste the plan into a worker
  prompt (Solo truncates large inputs). Give each worker the scratchpad key to
  read plus its bounded task.
- **Only `send_input` to a worker once it is idle.** Sending mid-work is lost.
- **Spawn is expensive and hard to undo.** Never spawn before the Phase 2 gate.
- **Ownership is law.** Every worker gets an explicit file-ownership boundary,
  is told other agents may be editing the repo, and is told NOT to revert
  unrelated changes.

## Phase 0 — Capture the plan

1. Derive the plan from THIS conversation (plus `$ARGUMENTS`). Do not ask the
   user to re-describe background already discussed.
2. Write a Solo scratchpad (`scratchpad_write`) containing:
   - **Goal** — one paragraph.
   - **Background** — enough that a fresh worker needs no further briefing.
   - **Lanes** — one section per lane, each with its own key (e.g. `lane-1`),
     holding that lane's task, acceptance/done condition, and any context.
   - **Ownership manifest** — a table of lane → files/dirs it owns. No overlaps.
   Keep it concise but self-contained.

## Phase 1 — Build the work graph

3. Create one Solo todo per lane (`todo_create`).
4. Set cross-lane dependencies with `todo_set_blockers` / `todo_add_blocker`.
5. Tag each todo `parallel` or `lead` (`todo_add_tag`). Exactly one lane may be
   the `lead` lane — work the lead keeps for itself.
6. Batch the independent todo/blocker/tag calls rather than one at a time.

## Phase 2 — Spawn gate (STOP for user)

7. Show the user a compact breakdown: the lanes, which are `parallel` vs `lead`,
   the blocker graph, and the ownership manifest.
8. **Wait for explicit "go" before spawning anything.** If the user adjusts
   lanes/ownership, update the scratchpad and todos, then re-show and re-gate.

## Phase 3 — Dispatch the first wave

9. Identify all unblocked `parallel` todos. Spawn a worker for each with
   `spawn_agent`, batched. Each spawn prompt is small and self-contained:
   - "You own lane `<key>`. Read scratchpad `<name>` key `<key>` for full context."
   - The bounded task and its done condition.
   - The exact files/dirs it owns (from the manifest).
   - "Other agents may be editing this repo concurrently. Do NOT revert or touch
     changes outside your owned files."
   - "When done, report: changed files, tests run + results, blockers hit,
     remaining risk."
10. Record spawned worker → todo mapping in the scratchpad.

## Phase 4 — Supervise (loop until no active workers)

11. Arm Solo's built-in idle-wake: `timer_fire_when_idle_any` across the active
    workers. Wake the instant any worker goes idle — do not poll on a clock.
12. On each wake, for the idle worker:
    a. Inspect it (`get_process_status` / `get_process_output`).
    b. Harvest its handoff: changed files, tests run, blockers, remaining risk.
       Save the useful parts to that worker's todo (`todo_comment_create` /
       `todo_update`); mark the todo complete if done (`todo_complete`).
    c. Reconcile the scratchpad and blockers with what was learned; update the
       ownership manifest if lanes shifted (`scratchpad_append_section` /
       `todo_remove_blocker`).
    d. Dispatch the next newly-unblocked wave (Phase 3 rules).
13. While workers run, the lead works its own `lead` todo — do not sit idle.
14. **Check-ins:** after each wave, give the user ONE short status line (what
    finished, what dispatched, what's blocked). Interrupt immediately — outside
    the wave summary — only when you need user input or an integration decision
    is ready.
15. Re-arm the idle-wake and continue until no workers remain active.

## Phase 5 — Close out

16. For each finished worker whose handoff is captured in its todo, close it
    (`close_process`). Keep any worker still producing useful work.
17. If a worker has descendants (spawned its own children), inspect them
    (`list_processes`) before closing the group.
18. Final report to the user: what shipped per lane, tests run, open risks, and
    anything still needing an integration decision.

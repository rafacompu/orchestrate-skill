---
name: orchestrate
description: "Turn the plan just discussed in this conversation into a Solo-orchestrated fan-out: scratchpad plan, todos with blockers, parallel worker agents on isolated branches, idle-wake supervision, and clean close-out. Re-runnable via --resume. Invoke explicitly with /orchestrate after a solution has been discussed."
argument-hint: "(optional) steering notes, or --resume"
disable-model-invocation: true
---

# /orchestrate — plan → Solo fan-out → close-out

The user typed `/orchestrate`. A solution has already been discussed in this
conversation. Turn that discussion into a supervised, parallel Solo workstream.
`$ARGUMENTS`, if present, is extra steering to fold into the plan — except the
literal `--resume`, which triggers the resume path below.

Use the Solo MCP tools throughout (`mcp__solo__*`). If Solo project scope is
unset, call `list_projects` / `select_project` first.

## Core principles (the reasoning behind the rules)

- **Brief via scratchpad key.** Workers read their full context from the
  scratchpad. Spawn prompts stay tiny — never paste the plan into a worker
  prompt, because Solo truncates large inputs and a half-delivered brief is
  worse than a pointer. Give each worker the scratchpad key to read plus its
  bounded task.
- **Only `send_input` to a worker once it is idle.** Input sent mid-work is lost.
- **Isolation is physical, not verbal.** Each worker runs on its own git branch
  (see Phase 3). Telling a worker "don't touch other files" is a hope; a
  separate branch is a guarantee — an interrupted or wrong lane is deleted and
  re-run with nothing half-done left on the main line.
- **Merged is the only durable "done."** A worker reporting success is a claim,
  not a fact. A lane is done when its branch merges cleanly into the
  integration branch — nothing earlier.
- **Spawn is expensive and hard to undo.** Never spawn before the Phase 2 gate,
  and never exceed the concurrency cap (default 4 concurrent workers).

## Resume path (`--resume`)

If `$ARGUMENTS` is `--resume`, do NOT re-plan. Instead:
1. Read the existing scratchpad and `todo_list` for this workstream.
2. Reconcile: mark which lanes are merged (done), which branches exist but are
   unmerged (in-flight — treat as disposable, re-dispatchable), which are still
   pending/blocked.
3. Report that reconstructed state to the user, then continue from Phase 4
   (supervise) for anything unfinished. This works because the scratchpad and
   todos are durable — they outlive the session that created them.

## Phase 0 — Capture the plan

1. Derive the plan from THIS conversation (plus `$ARGUMENTS`). Do not ask the
   user to re-describe background already discussed.
2. Write a Solo scratchpad (`scratchpad_write`) containing:
   - **Goal** — one paragraph.
   - **Background** — enough that a fresh worker needs no further briefing.
   - **Lanes** — one section per lane, each with its own key (e.g. `lane-1`),
     holding that lane's task, acceptance/done condition, and context.
   - **Dependencies** — for each lane, written explicitly: "reads the output of
     `lane-X`" or "none". Blockers in Phase 1 are DERIVED from these written
     lines, not guessed — so the dependency graph is auditable.
   - **Ownership manifest** — a table of lane → files/dirs it owns. No overlaps.
   Keep it concise but self-contained.

## Phase 1 — Build the work graph

3. Create one Solo todo per lane (`todo_create`).
4. Set blockers from the written Dependencies lines (`todo_set_blockers` /
   `todo_add_blocker`) — a lane that reads `lane-X`'s output is blocked on it.
5. Tag each todo `parallel` or `lead` (`todo_add_tag`). Exactly one lane may be
   the `lead` lane — work the lead keeps for itself.
6. Batch the independent todo/blocker/tag calls rather than one at a time.

## Phase 2 — Pre-spawn checks + gate (STOP for user)

7. **Ownership collision check.** Before showing the gate, verify no two lanes
   claim overlapping files, and that each claimed path actually exists in the
   repo (or is a declared new file). A lane built against a path that does not
   exist, or two lanes editing the same file, is the cheapest bug to catch now
   and the most expensive to catch after agents have run. Surface any conflict
   and halt for the user to resolve.
8. Show the user a compact breakdown: the lanes, which are `parallel` vs `lead`,
   the blocker graph, the ownership manifest, and the concurrency cap.
9. **Wait for explicit "go" before spawning anything.** If the user adjusts
   lanes/ownership, update the scratchpad and todos, then re-run step 7 and
   re-gate.

## Phase 3 — Dispatch a wave

10. Identify unblocked `parallel` todos. Spawn workers up to the concurrency cap
    (default 4; honor any cap the user set at the gate) with `spawn_agent`,
    batched. Each spawn prompt is small and self-contained:
    - "You own lane `<key>`. Read scratchpad `<name>` key `<key>` for full context."
    - "**Create and work on git branch `<lane-branch>`. Do all your edits there;
      do not commit to the integration branch.**" (one branch per lane)
    - The bounded task and its done condition.
    - The exact files/dirs it owns (from the manifest).
    - "Other agents are working concurrently on their own branches. Stay within
      your owned files; do not revert or touch anything outside them."
    - "When done, report: branch name, changed files, tests run + results,
      blockers hit, remaining risk."
11. Record spawned worker → todo → branch mapping in the scratchpad.

## Phase 4 — Supervise (loop until no active workers)

12. Arm Solo's built-in idle-wake: `timer_fire_when_idle_any` across the active
    workers. Wake the instant any worker goes idle — do not poll on a clock.
13. On each wake, for the idle worker:
    a. Inspect it (`get_process_status` / `get_process_output`).
    b. Harvest its handoff (branch, changed files, tests, blockers, risk) and
       save the useful parts to its todo (`todo_comment_create` / `todo_update`).
    c. **Verify "done" = merged.** Attempt to merge the lane's branch into the
       integration branch. Only on a clean merge mark the todo complete
       (`todo_complete`). A conflict or failed check means not-done — see retry.
    d. Reconcile the scratchpad and blockers with what was learned
       (`scratchpad_append_section` / `todo_remove_blocker`).
    e. Dispatch the next unblocked wave (Phase 3 rules, respecting the cap).
14. **Retry policy.** If a lane fails or its merge is unclean, retry it ONCE on
    a fresh branch. If it fails for a reason a re-run won't fix (a missing
    dependency, a spec defect, repeated identical failure), mark it `no-retry`
    in the scratchpad and surface it to the user instead of burning another
    slot on the same wall.
15. While workers run, the lead works its own `lead` todo — do not sit idle.
16. **Check-ins:** after each wave, give the user ONE short status line (what
    merged, what dispatched, what's blocked or retrying). Interrupt
    immediately — outside the wave summary — only when you need user input or an
    integration decision is ready.
17. Re-arm the idle-wake and continue until no workers remain active.

## Phase 5 — Close out

18. For each finished worker whose handoff is captured and whose branch is
    merged, close it (`close_process`). Keep any worker still producing useful
    work.
19. If a worker has descendants (spawned its own children), inspect them
    (`list_processes`) before closing the group.
20. Delete the merged lane branches; leave any `no-retry` branch intact for the
    user to inspect.
21. Final report to the user: what merged per lane, tests run, open risks,
    anything on `no-retry`, and anything still needing an integration decision.

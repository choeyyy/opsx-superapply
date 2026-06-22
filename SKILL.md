---
name: opsx-superapply
description: >-
  Subagent-driven implementation for OpenSpec changes. Dispatches a fresh subagent per task
  with two-stage review (spec compliance then code quality). Use instead of /opsx-apply when
  you want autonomous multi-agent execution with quality gates.
---

Subagent-driven implementation for OpenSpec changes. Each task is executed by an isolated subagent, then reviewed in two stages (spec compliance, then code quality) before marking complete.

**Input**: Optionally specify a change name. If omitted, auto-select or prompt. User may override model per role in conversation (e.g., "审查用 haiku").

**Steps**

1. **Select the change**

   If a name is provided, use it. Otherwise:
   - Infer from conversation context
   - Auto-select if only one active change exists
   - If ambiguous, run `openspec list --json` and use **AskQuestion tool** to let user select

   Announce: "Using change: \<name\> (super-apply mode)"

2. **Get status and context**

   ```bash
   openspec status --change "<name>" --json
   openspec instructions apply --change "<name>" --json
   ```

   Handle states:
   - `state: "blocked"` -- show message, suggest `/openspec-propose`
   - `state: "all_done"` -- congratulate, suggest `/opsx-archive`
   - Otherwise -- continue

   Read every file path listed under `contextFiles`. Cache as `CHANGE_CONTEXT` (proposal excerpt, relevant spec content, design constraints) for injection into subagent prompts.

3. **Extract task list and detect checkpoint**

   Parse tasks file. Build TodoWrite list of all tasks.
   - Skip tasks already marked `- [x]` (checkpoint resume)
   - Start from first `- [ ]` task

   Display progress:
   ```
   ## Super Apply: <change-name> (schema: <schema-name>)
   Progress: N/M tasks | Resuming from task K (or: Starting fresh)
   Remaining: <task summaries>
   ```

4. **Task execution loop**

   Execute continuously. Do NOT pause between tasks to ask "continue?". Stop only when: all tasks done, BLOCKED that controller cannot resolve, or genuine ambiguity.

   For each pending task:

   a. Record baseline: `BASE_SHA=$(git rev-parse HEAD)`

   b. **Dispatch implementer** — Read `./implementer-prompt.md`, fill placeholders (`{TASK_FULL_TEXT}`, `{PROPOSAL_EXCERPT}`, `{SPEC_CONTENT}`, `{DESIGN_CONSTRAINTS}`, `{PROJECT_ROOT}`), dispatch via Task tool (`subagent_type: generalPurpose`, `run_in_background: false`). Do NOT pass `model` parameter unless user explicitly requested a specific model.

   c. **Handle implementer status:**
      - DONE -- proceed to spec review
      - DONE_WITH_CONCERNS -- assess concerns; correctness issues: address first; observations: note and proceed
      - NEEDS_CONTEXT -- provide context, re-dispatch
      - BLOCKED -- escalate: supplement context / re-dispatch with stronger model / split task / pause and suggest updating OpenSpec artifacts

   d. **Dispatch spec reviewer** — Read `./spec-reviewer-prompt.md`, fill placeholders (`{TASK_REQUIREMENTS}`, `{IMPLEMENTER_REPORT}`), dispatch via Task tool (`readonly: true`).
      - Pass → proceed to quality review
      - Fail → re-dispatch implementer to fix specific issues, then re-review. Loop until pass.

   e. **Dispatch quality reviewer** — Record `HEAD_SHA=$(git rev-parse HEAD)`. Read `./quality-reviewer-prompt.md`, fill placeholders (`{TASK_SUMMARY}`, `{TASK_REQUIREMENTS}`, `{BASE_SHA}`, `{HEAD_SHA}`), dispatch via Task tool (`readonly: true`).
      - Critical/Important issues → re-dispatch implementer to fix, then re-review
      - Minor only or clean → pass

   f. **Mark complete** — Update tasks file: `- [ ]` → `- [x]`. Update TodoWrite. Continue to next task.

5. **Final review** (if change has > 3 tasks)

   Dispatch quality reviewer over full git range (first task BASE_SHA → final HEAD_SHA) for integration consistency check.

6. **Show completion status**

   Display summary with per-task review results and review statistics.

**Output During Execution**

```
## Super Apply: <change-name> (schema: <schema-name>)

Task 3/7: <task description>
  Impl: DONE | Spec: Pass | Quality: Pass
  -> Task complete

Task 4/7: <task description>
  Impl: DONE | Spec: Fail -> fix -> Pass | Quality: Pass
  -> Task complete
```

**Output On Completion**

```
## Super Apply Complete

**Change:** <change-name>
**Schema:** <schema-name>
**Progress:** M/M tasks complete

| # | Task | Impl | Spec | Quality |
|---|------|------|------|---------|
| 1 | ... | DONE | Pass | Pass |
| 2 | ... | DONE | Fail->Pass | Pass |

**Review stats:** N spec fixes, N quality fixes, N total rounds

All tasks complete! Run `/opsx-archive` to archive this change.
```

**Output On Pause**

```
## Super Apply Paused

**Change:** <change-name>
**Progress:** K/M tasks complete

### Blocker
<description>

**Options:**
1. <option>
2. <option>
3. Other

What would you like to do?
```

**Guardrails**
- Execute continuously — do not pause between tasks to ask permission
- Context isolation — each subagent gets only current task + change context, not conversation history
- Two-stage review is mandatory — never skip, regardless of task simplicity
- Fix-then-re-review -- reviewer found issues, implementer fixes, reviewer re-reviews, no skipping
- Spec review before quality review -- confirm "built the right thing" before "built it well"
- Update artifacts on design flaws -- if implementation reveals spec/design issues, pause and suggest updates rather than forcing through
- Minimal changes — each task's code changes scoped to that task only
- Checkpoint resume — always check for `- [x]` tasks and skip them on re-invocation

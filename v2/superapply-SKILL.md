---
name: opsx-superapply
description: >-
  Parallel multi-agent implementation for OpenSpec changes. Dispatches one subagent per
  capability group with spec-review gates. Use with /opsx-superpropose for grouped tasks.
  Achieves faster execution through parallelism and fewer errors through independent review.
---

Parallel multi-agent implementation for OpenSpec changes. Each capability group is executed by an isolated subagent, then verified against its spec before proceeding.

**Input**: Optionally specify a change name. If omitted, auto-select or prompt.

**Prerequisites**: openspec CLI installed (`npm i -g @fission-ai/openspec`), project git-initialized, at least one active change with grouped tasks (from `/opsx-superpropose`).

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
   - `state: "blocked"` — show message, suggest `/opsx-superpropose`
   - `state: "all_done"` — congratulate, suggest `/opsx-archive`
   - Otherwise — continue

   Read every file path listed under `contextFiles` (proposal, design, specs).

3. **Parse groups from tasks.md**

   Read the tasks file. Parse `###` headers to identify capability groups.
   For each group:
   - `group.name` = the `###` header text (matches `specs/<name>/spec.md`)
   - `group.tasks` = the `- [ ]` / `- [x]` items under that header
   - `group.spec` = content of `specs/<group.name>/spec.md`
   - `group.done` = all tasks are `[x]`

   Skip groups where `group.done == true` (checkpoint resume).

   Build TodoWrite list of all groups.

4. **Determine execution order**

   Read each group's spec content. Identify file paths and modules mentioned.

   - If two groups' specs reference non-overlapping files → **parallel**
   - If group B references files that group A creates → **B depends on A**
   - If unclear → default to **sequential** (safe fallback)

   Display execution plan:
   ```
   ## Super Apply: <change-name>
   Progress: N/M groups | Resuming from group K (or: Starting fresh)

   Parallel: [group-a], [group-b]
   Then: [group-c] (after group-a)
   ```

5. **Execute groups**

   Execute continuously. Do NOT pause between groups to ask "continue?".

   For parallel groups — dispatch simultaneously with `run_in_background: true`:

   For each pending group:

   a. Record baseline: `BASE_SHA=$(git rev-parse HEAD)`

   b. **Dispatch implementer subagent** — Read `./implementer-prompt.md`, fill placeholders:
      - `{GROUP_TASKS}`: all tasks in this group (full text)
      - `{SPEC_CONTENT}`: full content of this group's spec.md
      - `{PROPOSAL_EXCERPT}`: proposal.md summary (why + capabilities)
      - `{DESIGN_CONSTRAINTS}`: design.md content (constraints and decisions)
      - `{PROJECT_ROOT}`: workspace root path
      - `{SCOPE_FILES}`: file paths mentioned in this group's spec

      Dispatch via Task tool (`subagent_type: generalPurpose`).
      For parallel groups: `run_in_background: true`
      For sequential groups: `run_in_background: false`

   c. **Handle implementer result:**
      - DONE — proceed to spec review
      - DONE_WITH_CONCERNS — assess; correctness issues: address first; observations: note and proceed
      - NEEDS_CONTEXT — provide context, re-dispatch (resume same subagent)
      - BLOCKED — escalate: provide more context / use stronger model / split group / pause

   d. **Dispatch spec-reviewer subagent** — Read `./spec-reviewer-prompt.md`, fill placeholders:
      - `{SPEC_CONTENT}`: full spec.md for this group
      - `{IMPLEMENTER_REPORT}`: what the implementer reported
      - `{GROUP_NAME}`: capability name

      Dispatch via Task tool (`subagent_type: generalPurpose`, `readonly: true`).
      - Pass → mark group complete
      - Fail → resume implementer subagent with reviewer feedback, then re-review

   e. **Mark group complete** — Update tasks.md: all `- [ ]` in this group → `- [x]`.
      Update TodoWrite. Continue to next group.

6. **Integration quality review** (after ALL groups complete)

   Record `HEAD_SHA=$(git rev-parse HEAD)`.
   Read `./quality-reviewer-prompt.md`, fill placeholders:
   - `{CHANGE_SUMMARY}`: change name + all group names
   - `{ALL_SPECS}`: concatenated content of all specs
   - `{FIRST_BASE_SHA}`: BASE_SHA from the first group
   - `{HEAD_SHA}`: current HEAD

   Dispatch via Task tool (`subagent_type: generalPurpose`, `readonly: true`).
   - Critical issues → resume relevant implementer to fix, then re-review
   - Minor or clean → pass

7. **Show completion status**

   ```
   ## Super Apply Complete

   **Change:** <change-name>
   **Progress:** M/M groups complete

   | # | Group | Impl | Spec Review | Status |
   |---|-------|------|-------------|--------|
   | 1 | ... | DONE | Pass | Complete |
   | 2 | ... | DONE | Pass | Complete |

   Integration review: Pass

   All groups complete! Run `/opsx-archive` to archive this change.
   ```

**Output On Pause**

```
## Super Apply Paused

**Change:** <change-name>
**Progress:** K/M groups complete

### Blocker
<description>

**Options:**
1. <option>
2. <option>
3. Other

What would you like to do?
```

**Guardrails**
- Execute continuously — do not pause between groups to ask permission
- Each subagent gets: its group's tasks + full spec + proposal/design context
- Spec review is mandatory — never skip, regardless of group simplicity
- Fix-then-re-review — reviewer found issues, implementer fixes, reviewer re-reviews
- Scope discipline — each implementer should only modify files mentioned in its spec
- Checkpoint resume — always check for completed groups (`[x]`) and skip them
- Report CLI errors verbatim — if openspec commands fail, surface exact error and halt
- Do not invent task content — all tasks come from the parsed tasks file
- Parallel safety — only dispatch groups in parallel if their specs reference non-overlapping files
- If parallel groups conflict (git merge issues), fall back to sequential re-execution

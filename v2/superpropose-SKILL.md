---
name: opsx-superpropose
description: >-
  Propose a new change with parallel-ready grouped tasks. Same as /opsx-propose but generates
  tasks.md grouped by capability (### headers matching spec folder names) for use with
  /opsx-superapply. Use when you want multi-agent parallel execution with review gates.
---

Propose a new change with parallel-ready task grouping.

I'll create a change with artifacts:
- proposal.md (what & why)
- design.md (constraints & decisions)
- specs/*/spec.md (detailed plan per capability)
- tasks.md (grouped by capability for parallel execution)

When ready to implement, run `/opsx-superapply`

---

**Input**: The user's request should include a change name (kebab-case) OR a description of what they want to build.

**Steps**

1. **If no clear input provided, ask what they want to build**

   Use the **AskQuestion tool** to ask:
   > "What change do you want to work on? Describe what you want to build or fix."

   From their description, derive a kebab-case name (e.g., "add user authentication" → `add-user-auth`).

   **IMPORTANT**: Do NOT proceed without understanding what the user wants to build.

2. **Create the change directory**
   ```bash
   openspec new change "<name>"
   ```
   This creates a scaffolded change at `openspec/changes/<name>/` with `.openspec.yaml`.

3. **Get the artifact build order**
   ```bash
   openspec status --change "<name>" --json
   ```
   Parse the JSON to get:
   - `applyRequires`: array of artifact IDs needed before implementation (e.g., `["tasks"]`)
   - `artifacts`: list of all artifacts with their status and dependencies

4. **Create artifacts in sequence until apply-ready**

   Use the **TodoWrite tool** to track progress through the artifacts.

   Loop through artifacts in dependency order (artifacts with no pending dependencies first):

   a. **For each artifact that is `ready` (dependencies satisfied)**:
      - Get instructions:
        ```bash
        openspec instructions <artifact-id> --change "<name>" --json
        ```
      - The instructions JSON includes:
        - `context`: Project background (constraints for you - do NOT include in output)
        - `rules`: Artifact-specific rules (constraints for you - do NOT include in output)
        - `template`: The structure to use for your output file
        - `instruction`: Schema-specific guidance for this artifact type
        - `outputPath`: Where to write the artifact
        - `dependencies`: Completed artifacts to read for context
      - Read any completed dependency files for context
      - Create the artifact file using `template` as the structure
      - Apply `context` and `rules` as constraints - but do NOT copy them into the file
      - Show brief progress: "Created <artifact-id>"

   b. **Continue until all `applyRequires` artifacts are complete**
      - After creating each artifact, re-run `openspec status --change "<name>" --json`
      - Check if every artifact ID in `applyRequires` has `status: "done"` in the artifacts array
      - Stop when all `applyRequires` artifacts are done

   c. **If an artifact requires user input** (unclear context):
      - Use **AskQuestion tool** to clarify
      - Then continue with creation

   d. **SUPER-PROPOSE ADDITION: When creating the `tasks` artifact**:

      Instead of generating a flat task list, group tasks by capability:

      - Read all completed spec files under `specs/*/spec.md`
      - For each spec (capability), gather the tasks that implement it
      - Write tasks.md using `###` headers matching spec folder names:

        ```markdown
        ### <spec-folder-name>
        - [ ] Task description 1
        - [ ] Task description 2

        ### <another-spec-folder>
        - [ ] Task description 3
        ```

      - Header names MUST exactly match the folder names under `specs/`
      - Each task should be concrete and actionable (file paths, class/method names)
      - Tasks within a group should be in logical execution order
      - If a group depends on another group's output, place it after that group in the file

5. **Show execution plan**

   After all artifacts are created, analyze the specs to present a parallel execution plan:

   - Read each spec's content to identify file paths and modules it touches
   - Groups whose specs reference non-overlapping files → mark as parallelizable
   - Groups that reference files from another group → mark as sequential (depends on that group)
   - Present to user:

   ```
   ## Execution Plan

   Parallel group 1: [spec-a], [spec-b]  (no file overlap)
   Sequential after group 1: [spec-c]    (depends on spec-a outputs)

   Confirm? Or adjust grouping.
   ```

   Wait for user confirmation before finishing.

6. **Show final status**
   ```bash
   openspec status --change "<name>"
   ```

**Output**

After completing all artifacts, summarize:
- Change name and location
- List of artifacts created with brief descriptions
- Execution plan (which groups parallel, which sequential)
- Prompt: "Run `/opsx-superapply` to start parallel implementation with review gates."

**Artifact Creation Guidelines**

- Follow the `instruction` field from `openspec instructions` for each artifact type
- The schema defines what each artifact should contain - follow it
- Read dependency artifacts for context before creating new ones
- Use `template` as the structure for your output file - fill in its sections
- **IMPORTANT**: `context` and `rules` are constraints for YOU, not content for the file
  - Do NOT copy `<context>`, `<rules>`, `<project_context>` blocks into the artifact
  - These guide what you write, but should never appear in the output

**Tasks Grouping Guidelines**

- Each `###` header MUST match a folder name under `specs/`
- Tasks should include concrete details: file paths, class names, method signatures
- Write tasks at interface-level precision (what to create, where, what signature — not full code)
- Order groups so dependent groups come after their dependencies
- If a capability is truly standalone, it can parallel with others

**Guardrails**
- Create ALL artifacts needed for implementation (as defined by schema's `apply.requires`)
- Always read dependency artifacts before creating a new one
- If context is critically unclear, ask the user — but prefer making reasonable decisions to keep momentum
- If a change with that name already exists, ask if user wants to continue it or create a new one
- Verify each artifact file exists after writing before proceeding to next
- The execution plan is a suggestion — user can override parallelism decisions
- Do NOT start implementation — this skill only produces the proposal and plan

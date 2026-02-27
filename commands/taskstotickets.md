---
description: Convert existing tasks into Jira tickets for the feature based on available design artifacts, with optional parent ticket for sub-task creation.
tools:
  - 'Atlassian/getAccessibleAtlassianResources'
  - 'Atlassian/getVisibleJiraProjects'
  - 'Atlassian/getJiraIssue'
  - 'Atlassian/getJiraProjectIssueTypesMetadata'
  - 'Atlassian/createJiraIssue'
  - 'Atlassian/updateJiraIssue'
  - 'Atlassian/searchJiraIssuesUsingJql'
scripts:
  sh: ../scripts/bash/check-prerequisites.sh
  ps: ../scripts/powershell/check-prerequisites.ps1
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

The user may optionally provide a **parent ticket key** (e.g., `PROJ-123`). When provided, all tasks are created as Sub-tasks under that parent.

## Outline

1. Run `{SCRIPT} --json --require-tasks --include-tasks` from repo root and parse FEATURE_DIR and AVAILABLE_DOCS list. All paths must be absolute. For single quotes in args like "I'm Groot", use escape syntax: e.g 'I'\''m Groot' (or double-quote if possible: "I'm Groot").
1. From the executed script, extract the path to **tasks.md** and read it.
1. Get the Jira Cloud ID by calling `getAccessibleAtlassianResources`.

> [!CAUTION]
> ONLY PROCEED TO NEXT STEPS IF A VALID JIRA CLOUD RESOURCE IS RETURNED

4. **Determine the project key**:
   - If the user provided a parent ticket key (e.g., `PROJ-123`), extract the project key from it (the part before the hyphen: `PROJ`).
   - If no parent ticket was provided, call `getVisibleJiraProjects` to list available projects and confirm the target project with the user before proceeding.

5. **Validate parent ticket** (if provided):
   - Call `getJiraIssue` with the parent ticket key to confirm it exists.
   - If the parent ticket does not exist, **STOP** and report the error.

6. **Estimate and set story points on parent ticket** (if parent ticket provided):

   **Read current points**: From the parent ticket fetched in step 5, extract the story points field (may be `story_points`, `customfield_10016`, or another custom field — use the value returned by `getJiraIssue`).

   **Estimate effort (AI-assisted baseline)**: Analyze ALL tasks from tasks.md to produce an estimate that assumes the developer is using Spec Kit and an AI coding agent to implement. Use the following heuristic:

   | Signal | Points contribution |
   |--------|---------------------|
   | **Total incomplete phases** | Base: 1 pt per phase |
   | **Task volume** (incomplete tasks across all phases) | 1–10 tasks: +0, 11–20: +1, 21–30: +2, 31+: +3 |
   | **Integration complexity** — tasks involving external services, databases, auth, or third-party APIs | +1 per distinct integration |
   | **Testing depth** — dedicated test phases, contract tests, E2E, or performance benchmarks | +1 if significant test coverage required |
   | **Novelty / research** — tasks referencing research.md, spikes, or unfamiliar tech | +1 if present |
   | **Data model complexity** — >5 entities, migrations, or seed data | +1 if present |

   Snap the raw total to the nearest Fibonacci value: **1, 2, 3, 5, 8, 13, 21**.

   > The heuristic intentionally deflates compared to manual estimation because AI handles boilerplate, scaffolding, test generation, and repetitive implementation. Human effort concentrates on review, integration debugging, and design decisions.

   **Apply or suggest**:

   - **Parent has NO story points (null/0)**:
     - Present the estimate with a brief breakdown of contributing signals.
     - Ask the user to confirm or adjust before setting.
     - On confirmation, call `updateJiraIssue` to set the story points on the parent ticket.

   - **Parent already HAS story points**:
     - Compare the existing value against the AI-assisted estimate.
     - **If they match** (same Fibonacci value): report "Current estimate aligns with AI-assisted analysis" and move on.
     - **If they differ**: present both values with reasoning:

       ```text
       Current story points: 8
       AI-assisted estimate: 5

       Reasoning: 6 phases, 18 tasks, 1 external integration (Redis).
       The current estimate may reflect manual implementation effort.
       With Spec Kit + AI, scaffolding and boilerplate are handled automatically,
       reducing hands-on effort to review, integration, and design decisions.

       → Suggest updating to 5? (yes / no / custom value)
       ```

     - If the user confirms a change, call `updateJiraIssue` to update. Otherwise, keep the existing value.

   **If no parent ticket was provided**: Skip this step entirely (standalone Task tickets don't need a parent-level estimate).

7. **Discover issue types**: Call `getJiraProjectIssueTypesMetadata` for the target project to confirm available issue type names (e.g., `Task`, `Sub-task`). Use the exact names returned by the API — do not hardcode type names.

8. **Discover naming convention** (when parent ticket is provided):
   - Search for existing sub-tasks under the parent using JQL: `parent = <PARENT_KEY> ORDER BY key ASC`
   - If sibling tickets exist, analyze their summary titles to extract the naming pattern (prefix style, voice, length, structure) and match it for all new tickets.
   - If no sibling tickets exist, use the default convention below.

   **Default naming convention**:
   - Format: `Phase N: Concise action-oriented description`
   - Keep the `Phase N:` prefix — it preserves execution order at a glance
   - **NEVER** copy the raw phase heading from tasks.md verbatim (e.g., `Phase 3: User Story 1 — Browse Filing Guides by State`) — always rewrite the part after `Phase N:` into a concise deliverable summary

   **Cohesive description rules** (apply to ALL tickets in the set):
   - Start with an action verb after `Phase N:` (Create, Implement, Add, Verify, Harden, etc.)
   - Describe the **outcome or deliverable**, not internal artifacts (prefer "Implement typeahead endpoint with cached prefix search" over "Create DTOs and service interfaces")
   - Use consistent voice and abstraction level across all tickets — if one says "Implement X", don't have another say "T012-T014 for US3"
   - Keep the description portion to one short phrase (under ~10 words after `Phase N:`)
   - No feature name suffix, no task IDs, no "User Story N —" labels in the summary

9. **Group tasks by phase**: Parse tasks.md by phase headers (`## Phase N: ...`). Each phase becomes one Jira ticket. For each phase that contains at least one incomplete task (`- [ ]`):
   - **Summary**: A concise title following the naming convention discovered in step 8.
   - **Description** (Markdown): Include the phase goal/purpose, checkpoint criteria (if present), and the full list of task lines in that phase (with their IDs, `[P]`/`[US#]` markers, and file paths). This gives the implementer all the context they need within the ticket.
   - **Issue type**: If a parent ticket was provided, use the Sub-task type. Otherwise, use the Task type.
   - **Parent**: Set to the parent ticket key when provided.

> [!CAUTION]
> SKIP any phase where ALL tasks are already completed (`- [x]` or `- [X]`). A phase with a mix of complete and incomplete tasks still gets a ticket (the description should note which tasks are already done).

10. **Persist mapping**: Write a `jira-map.md` file to FEATURE_DIR (alongside tasks.md, plan.md, etc.) with the following format:

   ```markdown
   # Jira Task Map

   **Project**: PROJ | **Parent**: PROJ-123 (or "None") | **Created**: YYYY-MM-DD

   | Jira Key | Task IDs | Phase |
   |----------|----------|-------|
   | PROJ-456 | T006, T007, T008, T009, T010, T011, T012, T013, T014 | Phase 3: User Story 1 — Browse Filing Guides |
   | PROJ-457 | T015, T016, T017, T018 | Phase 4: User Story 2 — State-Scoped Routes |
   ```

   - Use the actual project key, parent ticket (or "None"), and today's date.
   - Include one row per created ticket (one per phase).
   - Task IDs column lists all tasks in that phase, comma-separated.
   - This file is consumed by `/speckit.jira.implement` to scope implementation to individual tickets.

11. **Report**: After all tickets are created and the mapping is saved, output a summary:
   - Path to `jira-map.md`
   - One line per ticket: Jira key → phase name and task count (e.g., `PROJ-456 → Phase 3: US1 (9 tasks)`)
   - Total tickets created
   - Parent ticket (if used) and story points (set/updated/unchanged)
   - Link to the Jira board/project

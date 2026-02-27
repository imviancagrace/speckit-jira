---
description: Execute implementation scoped to a single Jira ticket's tasks using the jira-map.md mapping produced by /speckit.jira.taskstotickets.
tools:
  - 'Atlassian/getAccessibleAtlassianResources'
  - 'Atlassian/getJiraIssue'
scripts:
  sh: ../scripts/bash/check-prerequisites.sh
  ps: ../scripts/powershell/check-prerequisites.ps1
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

The user may optionally provide a **Jira ticket key** (e.g., `PROJ-456`). When provided, only the tasks mapped to that ticket are implemented. When omitted, the command auto-selects the next incomplete ticket.

## Outline

1. Run `{SCRIPT} --json --require-tasks --include-tasks` from repo root and parse FEATURE_DIR and AVAILABLE_DOCS list. All paths must be absolute. For single quotes in args like "I'm Groot", use escape syntax: e.g 'I'\''m Groot' (or double-quote if possible: "I'm Groot").

2. **Load jira-map.md**: Read `FEATURE_DIR/jira-map.md`.
   - If the file does not exist, **STOP** and report: "No Jira mapping found. Run `/speckit.jira.taskstotickets` first to create tickets and generate the mapping."

3. **Determine target ticket**:
   - If the user provided a Jira ticket key, look it up in the `jira-map.md` table to find the matching Task IDs for that phase.
     - If the key is not found in the mapping, **STOP** and report the error with the list of available ticket keys.
   - If no ticket key was provided:
     - Cross-reference each ticket's task IDs in `jira-map.md` against `tasks.md` completion status (`- [ ]` vs `- [x]`).
     - Display a status table of all mapped tickets with per-ticket progress:

       ```text
       | Jira Key   | Task IDs         | Phase                      | Progress | Status      |
       |------------|------------------|----------------------------|----------|-------------|
       | PROJ-450   | T001, T002, T003 | Phase 1: Setup             | 3/3      | ✓ Done      |
       | PROJ-451   | T004, T005       | Phase 2: Foundational      | 2/2      | ✓ Done      |
       | PROJ-456   | T006–T014        | Phase 3: US1               | 0/9      | ○ Next      |
       | PROJ-457   | T015–T018        | Phase 4: US2               | 0/4      | ○ Pending   |
       ```

     - Auto-select the first ticket with incomplete tasks (by phase order in tasks.md) and confirm with the user before proceeding.

4. **Check checklists status** (if FEATURE_DIR/checklists/ exists):
   - Scan all checklist files in the checklists/ directory
   - For each checklist, count:
     - Total items: All lines matching `- [ ]` or `- [X]` or `- [x]`
     - Completed items: Lines matching `- [X]` or `- [x]`
     - Incomplete items: Lines matching `- [ ]`
   - Create a status table:

     ```text
     | Checklist | Total | Completed | Incomplete | Status |
     |-----------|-------|-----------|------------|--------|
     | ux.md     | 12    | 12        | 0          | ✓ PASS |
     | test.md   | 8     | 5         | 3          | ✗ FAIL |
     | security.md | 6   | 6         | 0          | ✓ PASS |
     ```

   - Calculate overall status:
     - **PASS**: All checklists have 0 incomplete items
     - **FAIL**: One or more checklists have incomplete items

   - **If any checklist is incomplete**:
     - Display the table with incomplete item counts
     - **STOP** and ask: "Some checklists are incomplete. Do you want to proceed with implementation anyway? (yes/no)"
     - Wait for user response before continuing
     - If user says "no" or "wait" or "stop", halt execution
     - If user says "yes" or "proceed" or "continue", proceed to step 5

   - **If all checklists are complete**:
     - Display the table showing all checklists passed
     - Automatically proceed to step 5

5. Load and analyze the implementation context:
   - **REQUIRED**: Read tasks.md for the complete task list and execution plan
   - **REQUIRED**: Read plan.md for tech stack, architecture, and file structure
   - **IF EXISTS**: Read data-model.md for entities and relationships
   - **IF EXISTS**: Read contracts/ for API specifications and test requirements
   - **IF EXISTS**: Read research.md for technical decisions and constraints
   - **IF EXISTS**: Read quickstart.md for integration scenarios

6. **Project Setup Verification**:
   - **REQUIRED**: Create/verify ignore files based on actual project setup:

   **Detection & Creation Logic**:
   - Check if the following command succeeds to determine if the repository is a git repo (create/verify .gitignore if so):

     ```sh
     git rev-parse --git-dir 2>/dev/null
     ```

   - Check if Dockerfile* exists or Docker in plan.md → create/verify .dockerignore
   - Check if .eslintrc* exists → create/verify .eslintignore
   - Check if eslint.config.* exists → ensure the config's `ignores` entries cover required patterns
   - Check if .prettierrc* exists → create/verify .prettierignore
   - Check if .npmrc or package.json exists → create/verify .npmignore (if publishing)
   - Check if terraform files (*.tf) exist → create/verify .terraformignore
   - Check if .helmignore needed (helm charts present) → create/verify .helmignore

   **If ignore file already exists**: Verify it contains essential patterns, append missing critical patterns only
   **If ignore file missing**: Create with full pattern set for detected technology

   **Common Patterns by Technology** (from plan.md tech stack):
   - **Node.js/JavaScript/TypeScript**: `node_modules/`, `dist/`, `build/`, `*.log`, `.env*`
   - **Python**: `__pycache__/`, `*.pyc`, `.venv/`, `venv/`, `dist/`, `*.egg-info/`
   - **Java**: `target/`, `*.class`, `*.jar`, `.gradle/`, `build/`
   - **C#/.NET**: `bin/`, `obj/`, `*.user`, `*.suo`, `packages/`
   - **Go**: `*.exe`, `*.test`, `vendor/`, `*.out`
   - **Ruby**: `.bundle/`, `log/`, `tmp/`, `*.gem`, `vendor/bundle/`
   - **PHP**: `vendor/`, `*.log`, `*.cache`, `*.env`
   - **Rust**: `target/`, `debug/`, `release/`, `*.rs.bk`, `*.rlib`, `*.prof*`, `.idea/`, `*.log`, `.env*`
   - **Kotlin**: `build/`, `out/`, `.gradle/`, `.idea/`, `*.class`, `*.jar`, `*.iml`, `*.log`, `.env*`
   - **C++**: `build/`, `bin/`, `obj/`, `out/`, `*.o`, `*.so`, `*.a`, `*.exe`, `*.dll`, `.idea/`, `*.log`, `.env*`
   - **C**: `build/`, `bin/`, `obj/`, `out/`, `*.o`, `*.a`, `*.so`, `*.exe`, `Makefile`, `config.log`, `.idea/`, `*.log`, `.env*`
   - **Swift**: `.build/`, `DerivedData/`, `*.swiftpm/`, `Packages/`
   - **R**: `.Rproj.user/`, `.Rhistory`, `.RData`, `.Ruserdata`, `*.Rproj`, `packrat/`, `renv/`
   - **Universal**: `.DS_Store`, `Thumbs.db`, `*.tmp`, `*.swp`, `.vscode/`, `.idea/`

   **Tool-Specific Patterns**:
   - **Docker**: `node_modules/`, `.git/`, `Dockerfile*`, `.dockerignore`, `*.log*`, `.env*`, `coverage/`
   - **ESLint**: `node_modules/`, `dist/`, `build/`, `coverage/`, `*.min.js`
   - **Prettier**: `node_modules/`, `dist/`, `build/`, `coverage/`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`
   - **Terraform**: `.terraform/`, `*.tfstate*`, `*.tfvars`, `.terraform.lock.hcl`
   - **Kubernetes/k8s**: `*.secret.yaml`, `secrets/`, `.kube/`, `kubeconfig*`, `*.key`, `*.crt`

7. **Filter tasks to ticket scope**: From the full tasks.md, extract ALL tasks whose Task IDs appear in the target ticket's row in `jira-map.md`. These tasks belong to the same phase, so preserve their original execution order and dependency/parallel rules from tasks.md. Before executing, verify that all tasks from earlier phases (earlier tickets in `jira-map.md`) are already completed (`- [x]`). If a prerequisite phase has incomplete tasks, **STOP** and report which ticket/phase must be finished first.

8. Execute the filtered task(s) following the implementation rules:
   - **Respect dependencies**: Run sequential tasks in order, parallel tasks [P] can run together
   - **Follow TDD approach**: Execute test tasks before their corresponding implementation tasks
   - **File-based coordination**: Tasks affecting the same files must run sequentially
   - **Validation checkpoints**: Verify task completion before reporting

9. Implementation execution rules:
   - **Setup first**: Initialize project structure, dependencies, configuration
   - **Tests before code**: If you need to write tests for contracts, entities, and integration scenarios
   - **Core development**: Implement models, services, CLI commands, endpoints
   - **Integration work**: Database connections, middleware, logging, external services
   - **Polish and validation**: Unit tests, performance optimization, documentation

10. Progress tracking and error handling:
    - Report progress after each completed task
    - Halt execution if any non-parallel task fails
    - For parallel tasks [P], continue with successful tasks, report failed ones
    - Provide clear error messages with context for debugging
    - Suggest next steps if implementation cannot proceed
    - **IMPORTANT** For completed tasks, make sure to mark the task off as [X] in the tasks file.

11. Completion validation:
    - Verify all tasks for the target ticket's phase are completed
    - Check that implemented features match the original specification
    - Validate that tests pass and coverage meets requirements
    - Confirm the implementation follows the technical plan
    - Report final status with:
      - Jira ticket key, phase name, and count of tasks completed (e.g., `PROJ-456 — Phase 3: US1 — 9/9 tasks`)
      - Summary of work done
      - Suggested next ticket/phase to implement (from jira-map.md)

Note: This command requires `jira-map.md` in the feature directory. If missing, run `/speckit.jira.taskstotickets` first. For full (unscoped) implementation, use `/speckit.implement` instead.

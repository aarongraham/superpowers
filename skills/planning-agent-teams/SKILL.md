---
name: planning-agent-teams
description: Use when user asks to create an agent team, coordinate multiple teammates, or when a task needs persistent agents that communicate with each other via TeamCreate and SendMessage
---

# Planning Agent Teams

## Overview

Bootstrap an agent team from a high-level user prompt. Ask quick clarifying questions, spawn a PM + Explorer in parallel, let the PM interview the user to refine requirements, then design the full team with roles, phases, and task dependencies.

**This skill is about agent teams** (TeamCreate, SendMessage, shared task lists, persistent teammates). If the task just needs parallel subagents that report back results, use `superpowers:dispatching-parallel-agents` instead.

**Announce at start:** "I'm using the planning-agent-teams skill to design your agent team."

## When to Use

**Use when:**
- User says "create a team", "agent team", "spawn teammates"
- Task needs persistent agents that communicate with each other
- Work has multiple phases with handoffs between specialists
- Parallel exploration where agents need to challenge or build on each other's findings

**Don't use when:**
- Task needs parallel subagents that just report back (use `superpowers:dispatching-parallel-agents`)
- Task is sequential and fits in one session (use `superpowers:subagent-driven-development`)
- Simple focused tasks where only the result matters (use Task tool directly)

## Step 1: Quick Triage

Ask the user 1-2 questions to determine the work category:

1. **What kind of work?** (QA, feature build, refactoring, debugging, research/review)
2. **Any constraints?** (tech stack, time, scope, specific areas to focus on)

Don't over-interview here — the PM agent will dig deeper.

## Step 2: Spawn Wave 0 — PM + Explorer

Spawn both in parallel immediately after triage.

**Explorer agent:**
- Read-only, analyzes the codebase
- Invokes `superpowers:systematic-debugging`
- Reports findings to lead: architecture, key files, potential issues
- Shuts down after reporting

**PM agent:**
- Read-only, interviews the user
- Invokes `superpowers:brainstorming`
- Incorporates Explorer's findings when they arrive (lead forwards them)
- After the interview, PM proposes recommended roles from the Role Catalog (see below)
- **PM must ask the user which roles they want** — present the recommendations, let the user add, remove, or adjust
- Reports final requirements + approved role list to lead

```
Task(
  subagent_type: "general-purpose",
  name: "explorer",
  team_name: "<team>",
  mode: "plan"  // read-only
)

Task(
  subagent_type: "general-purpose",
  name: "pm",
  team_name: "<team>",
  mode: "default"  // needs to interact with user
)
```

## Step 3: Design the Team

After PM reports back, the lead:

1. Selects roles from the Role Catalog based on PM's approved list
2. Designs phases with spawn waves (not all agents at once)
3. Creates the task list with dependencies using TaskCreate
4. Presents the full plan to the user:

```
## Agent Team Plan: <team-name>

### Roles
- PM (done) — requirements gathered
- Explorer (done) — codebase analyzed
- Architect — will design implementation plan
- Implementer — will build using TDD
- Verifier — will run quality suite

### Phases
1. Design: Architect plans the approach (blocked by: PM + Explorer)
2. Build: Implementer executes the plan (blocked by: Architect)
3. Verify: Verifier runs test suite (blocked by: Implementer)
4. Ship: Lead does final review + finishing-a-development-branch

### Task List
[numbered tasks with dependencies]

---
Approve this plan? Want to adjust roles or phases?
```

**Wait for user approval before spawning.**

## Step 4: Execute in Waves

Spawn teammates in waves — don't launch everyone at once.

- **Wave 0:** PM + Explorer (already done)
- **Wave 1:** Roles that can start immediately (e.g., Architect)
- **Wave 2:** Roles that depend on Wave 1 output (e.g., Implementer)
- **Wave 3:** Roles that verify Wave 2 output (e.g., Verifier, Reviewer)

Use **delegate mode** (Shift+Tab) to keep the lead focused on coordination.

Shut down teammates as they complete their work — don't leave idle agents running.

## Step 5: Coordinator Decision Loop

After each wave or when a teammate goes idle:

```
1. Check TaskList for completed/blocked tasks
2. Read messages from teammates
3. Decide:
   a. New finding/bug       → TaskCreate, assign to appropriate role
   b. Simple one-liner fix  → Lead fixes directly
   c. Out of scope          → Note for user, don't act
   d. Teammate done + work  → Assign next task
   e. Teammate done + idle  → SendMessage shutdown_request
   f. All done              → Final review + finishing-a-development-branch
4. Verifier finds failures  → Route back to Implementer
```

## Step 6: Cleanup

After all work is complete:

1. Shut down all remaining teammates
2. Invoke `superpowers:finishing-a-development-branch`
3. TeamDelete to clean up team resources

## Role Catalog

Select roles based on the work category. Each role has defined permissions, superpowers, and output expectations.

### PM
- **Purpose:** Interview user, produce requirements/spec
- **Permissions:** Read-only
- **Superpowers:** `superpowers:brainstorming`
- **Output:** Requirements doc + recommended role list (approved by user)
- **Lifecycle:** Wave 0, shuts down after requirements delivered

### Explorer
- **Purpose:** Analyze codebase architecture, identify key files, report findings
- **Permissions:** Read-only
- **Superpowers:** `superpowers:systematic-debugging`
- **Output:** Structured findings (architecture overview, key files, potential issues)
- **Lifecycle:** Wave 0, shuts down after report delivered

### Architect
- **Purpose:** Design implementation plan from PM's spec + Explorer's findings
- **Permissions:** Read-only (plan mode required — lead approves plan before execution)
- **Superpowers:** `superpowers:writing-plans`
- **Output:** Implementation plan with tasks, dependencies, and file paths
- **Lifecycle:** Wave 1, shuts down after plan approved

### Implementer
- **Purpose:** Build features or fix bugs using TDD
- **Permissions:** Read-write (`bypassPermissions` or `default`)
- **Superpowers:** `superpowers:test-driven-development`
- **Output:** Working code with tests, committed to branch
- **Lifecycle:** Wave 2, shuts down when all assigned tasks complete

### Verifier
- **Purpose:** Run test suites, linters, type checks — report only, never fix
- **Permissions:** Read-only (runs commands but doesn't edit files)
- **Superpowers:** `superpowers:verification-before-completion`
- **Output:** Pass/fail report with specific failures listed
- **Lifecycle:** Wave 3, shuts down after report. Failures route back to Implementer.

### Browser QA
- **Purpose:** Verify specific fixes/features work in the UI
- **Permissions:** Read-only
- **Skills:** `agent-browser`, `superpowers:verification-before-completion`
- **Output:** Targeted pass/fail results with screenshots
- **Lifecycle:** Wave 2-3, verifies after Implementer delivers fixes

### UX Explorer
- **Purpose:** Exploratory testing — discover bugs, papercuts, broken UI, evaluate general UX
- **Permissions:** Read-only
- **Skills:** `agent-browser`
- **Output:** UX audit — screenshots + descriptions grouped by severity (broken, papercut, suggestion)
- **Lifecycle:** Wave 1 (can run early, doesn't depend on code changes)

### Reviewer
- **Purpose:** Code review of all changes before shipping
- **Permissions:** Read-only
- **Superpowers:** `superpowers:requesting-code-review`
- **Output:** Review with severity ratings (Critical, Important, Minor)
- **Lifecycle:** Wave 3, after implementation complete

## Default Team Compositions by Category

Not every team uses every role. Start with the default for the work category, then let the PM refine with the user.

| Category | Default Roles | Typical Phases |
|----------|---------------|----------------|
| **QA / Bug Hunt** | PM, Explorer, UX Explorer, Browser QA, Implementer, Verifier | Explore > Triage > Fix > Verify |
| **Feature Build** | PM, Explorer, Architect, Implementer, Verifier, Reviewer | Design > Plan > Build > Review > Verify |
| **Refactoring** | PM, Explorer, Architect, Implementer, Verifier, Reviewer | Analyze > Plan > Refactor > Review > Verify |
| **Debugging** | PM, Explorer, Implementer, Verifier | Investigate > Hypothesize > Fix > Verify |
| **Research / Review** | PM, Explorer, Reviewer | Explore > Analyze > Synthesize |

## Spawn Prompt Guidelines

When spawning teammates, the prompt must include:

1. **Which superpowers/skills to invoke** (by name)
2. **Specific scope** — what files, what areas, what questions to answer
3. **Permission constraints** — read-only vs read-write
4. **Output format** — what to report and how (structured findings, pass/fail, etc.)
5. **Who to report to** — "message the lead when done"
6. **Codebase context** — key files, tech stack, known issues (from Explorer's findings)

## Red Flags

**Never:**
- Spawn all agents at once — use waves
- Let agents edit the same files — assign ownership
- Skip the PM interview — requirements prevent wasted work
- Skip user approval of the team plan
- Leave idle agents running — shut them down

**Always:**
- Let the PM ask the user which roles they want
- Give agents explicit permission constraints
- Forward Explorer findings to the PM
- Use delegate mode to keep the lead focused on coordination
- Clean up with TeamDelete when done

## Quick Reference

| Step | Action |
|------|--------|
| 1 | Quick triage: what kind of work + constraints |
| 2 | Spawn Wave 0: PM + Explorer in parallel |
| 3 | Design team: roles, phases, tasks from PM's findings |
| 4 | Present plan, wait for user approval |
| 5 | Execute in waves, coordinate with decision loop |
| 6 | Cleanup: shut down agents, finish branch, TeamDelete |

## Integration

**Pairs with:**
- **superpowers:brainstorming** — PM agent uses this for user interviews
- **superpowers:writing-plans** — Architect agent uses this for implementation plans
- **superpowers:finishing-a-development-branch** — Lead uses this to ship
- **superpowers:subagent-driven-development** — Alternative for single-session work

**Replaces (for team scenarios):**
- **superpowers:dispatching-parallel-agents** — That skill is for parallel subagents, not persistent teams

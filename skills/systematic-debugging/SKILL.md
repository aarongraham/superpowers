---
name: systematic-debugging
description: Use when encountering any bug, test failure, or unexpected behavior, before proposing fixes
---

# Systematic Debugging

## Overview

**ALWAYS find root cause before proposing fixes.**

This skill exists because the most dangerous debugging failure mode is: propose fix → test → fail → propose different fix → test → fail → repeat forever. Each untargeted fix wastes time and often introduces new problems.

**The Iron Law:**

```
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST
```

**Announce at start:** "I'm using the systematic-debugging skill to investigate this."

## When to Use

- ANY bug report
- ANY test failure
- ANY unexpected behavior
- ANY performance problem
- ANY "it worked before" situation

## Phase 1: Root Cause Investigation

**MUST complete before proposing any fix.**

### Step 1: Read Errors Carefully

```
STOP. Read the FULL error message.
- What file?
- What line?
- What operation failed?
- What was expected vs actual?
```

### Step 2: Reproduce Consistently

```bash
# Run the failing test/scenario
<exact command>

# Run it again - same failure?
<exact command>

# If different failures each time: that's a clue (race condition, state leak)
```

### Step 3: Check Recent Changes

```bash
# What changed recently?
git log --oneline -10
git diff HEAD~3

# Did the test ever pass? When did it start failing?
git log --oneline -- <test-file>
```

### Step 4: Gather Evidence in Multi-Component Systems

When the bug involves multiple components (frontend ↔ backend, service ↔ database, test ↔ implementation):

```
For EACH component in the chain:
1. Verify it receives correct input
2. Verify it produces correct output
3. Find the FIRST point where actual ≠ expected

The bug is at the boundary where correct → incorrect.
```

### Step 5: Trace Data Flow

```
Starting from the error:
1. What function threw?
2. What called that function?
3. What data was passed?
4. Where did that data come from?
5. At what point did the data become wrong?

Read the ACTUAL code at each step. Don't guess from function names.
```

## Phase 2: Pattern Analysis

### Step 1: Find Working Examples

```
Look for similar code/tests that DO work:
- Same pattern in different file?
- Same test for different feature?
- Documentation examples?
```

### Step 2: Compare Against Reference

Use supporting technique: `./root-cause-tracing.md`

```
Working example vs failing code:
- What's different?
- What assumptions differ?
- What dependencies differ?
```

### Step 3: Identify the Pattern

```
Is this a known pattern?
- Timing/race condition
- State leak between tests
- Wrong assertion (test bug, not code bug)
- Missing setup/teardown
- Dependency version mismatch
- Configuration drift
```

## Phase 3: Hypothesis & Testing

### The Rules

1. **Form a SINGLE hypothesis** based on evidence from Phases 1-2
2. **Test the hypothesis minimally** - smallest change that proves/disproves
3. **If disproved:** Return to Phase 1 with new evidence
4. **If confirmed:** Proceed to Phase 4

### Don't Know?

If you can't form a hypothesis after thorough investigation:

```
STOP. Say:
"I've investigated [what you checked] and found [what you found].
I don't have a clear hypothesis for why [problem].
Can you help me understand [specific question]?"
```

**Asking for help is better than guessing.**

## Phase 4: Implementation (Only After Root Cause Found)

1. **Create Failing Test Case** (REQUIRED)
   - Test that demonstrates the bug
   - Automated test if possible
   - One-off test script if no framework
   - MUST have before fixing
   - Use the `superpowers:test-driven-development` skill for writing proper failing tests

2. **Implement Single Fix**
   - Address the root cause identified
   - One change at a time
   - Don't "fix" things that aren't broken

3. **Verify Fix**
   - Failing test now passes?
   - Test passes now?
   - No other tests broken?
   - Issue actually resolved?

4. **If Fix Doesn't Work**
   - STOP
   - Return to Phase 1 with new evidence from the failed fix
   - Count fix attempts
   - **If ≥ 3: STOP and question the architecture (step 5 below)**
   - DON'T attempt Fix #4 without architectural discussion

5. **If 3+ Fixes Failed: Question Architecture**

   **Pattern indicating architectural problem:**
   - Each fix reveals new shared state/coupling/problem in different place
   - Fixes require "massive refactoring" to implement
   - Each fix creates new symptoms elsewhere

   **STOP and question fundamentals:**
   - Is this pattern fundamentally sound?
   - Are we "sticking with it through sheer inertia"?
   - Should we refactor architecture vs. continue fixing symptoms?

   **Discuss with your human partner before attempting more fixes**

   This is NOT a failed hypothesis - this is a wrong architecture.

## Red Flags - STOP and Follow Process

If you catch yourself thinking:

- "I think the problem might be..." (without evidence)
- "Let me just try..." (without root cause)
- "This usually fixes it..." (pattern matching without understanding)
- "Quick fix: just add a try-catch / null check / timeout"
- Changing multiple things at once
- Not reading the actual error message
- Not reproducing the issue first
- Assuming you know what code does without reading it
- Adding workarounds instead of fixing root cause
- Proposing solutions before tracing data flow
- **"One more fix attempt" (when already tried 2+)**
- **Each fix reveals new problem in different place**

**ALL of these mean: STOP. Return to Phase 1.**

**If 3+ fixes failed:** Question the architecture (see Phase 4.5)

## your human partner's Signals You're Doing It Wrong

| They say | You should |
|----------|-----------|
| "Did you actually read the error?" | Go back to Phase 1, Step 1 |
| "Why are you changing that?" | Explain your root cause hypothesis |
| "You already tried that" | STOP. Return to Phase 1 with all evidence |
| "Let's step back" | Return to Phase 1 completely |
| "That's not related" | Verify your hypothesis connects to root cause |

## Escalation Rules

| Situation | Action |
|-----------|--------|
| Can't reproduce | Document exact steps tried, ask for environment details |
| Root cause found but fix is complex | Document finding, propose approach, get approval before implementing |
| 3+ fix attempts failed | Architectural problem - discuss with human partner before more fixes |
| Involves unfamiliar system | Ask for help rather than guessing at architecture |

## Anti-Rationalization

| Thought | Reality |
|---------|---------|
| "I know this pattern" | Confirm with evidence before acting. |
| "It's probably just X" | "Probably" = you're guessing. Find evidence. |
| "Let me try a quick fix first" | Quick fixes without root cause create more bugs. |
| "I don't have time to investigate" | You don't have time for 5 failed fix attempts either. |
| "Multiple fixes at once saves time" | Can't isolate what worked. Causes new bugs. |
| "Reference too long, I'll adapt the pattern" | Partial understanding guarantees bugs. Read it completely. |
| "I see the problem, let me fix it" | Seeing symptoms ≠ understanding root cause. |
| "One more fix attempt" (after 2+ failures) | 3+ failures = architectural problem. Question pattern, don't fix again. |

## Quick Reference

```
Phase 1: INVESTIGATE (no fixing!)
  → Read error → Reproduce → Check changes → Trace data

Phase 2: ANALYZE
  → Find working example → Compare → Identify pattern

Phase 3: HYPOTHESIZE
  → Single hypothesis → Minimal test → Proved/disproved?

Phase 4: FIX (only after root cause confirmed)
  → Failing test → Single fix → Verify → Use defense-in-depth

If stuck: ASK FOR HELP
If 3+ failures: QUESTION ARCHITECTURE
```

## Integration

**Supporting techniques (bundled):**
- **`root-cause-tracing.md`** - Trace bugs backward through call stack
- **`defense-in-depth.md`** - Add validation at multiple layers after finding root cause
- **`condition-based-waiting.md`** - Replace arbitrary timeouts with condition polling

**Related skills:**
- **superpowers:test-driven-development** - For creating failing test case (Phase 4, Step 1)
- **superpowers:verification-before-completion** - Verify fix worked before claiming success

## Real-World Impact

Without this process: "Let me just try..." → 5 attempts → 2 hours wasted → finally read the error → fix in 5 minutes.

With this process: Read error → trace → find root cause → fix → 20 minutes total.

**The investigation feels slower. The total time is much faster.**

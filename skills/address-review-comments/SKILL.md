---
name: address-review-comments
description: Use when processing PR review comments, after receiving feedback on a pull request that needs to be triaged and addressed
user_invocable: address-review-comments
---

# Address Review Comments

## Overview

Fetch all PR comments, triage them, present the batch for approval, then execute fixes, replies, and thread resolution in one pass.

**Announce at start:** "I'm using the address-review-comments skill to process PR feedback."

## Step 1: Detect PR

If user provided a PR URL, extract owner/repo/number from it.

Otherwise, detect from current branch:

```bash
gh pr view --json number,url,headRepositoryOwner,headRepository
```

If no PR exists for this branch, stop: "No PR found for this branch."

## Step 2: Fetch All Comments

```bash
# Review comments (inline code comments) - includes thread IDs
gh api repos/{owner}/{repo}/pulls/{pr}/comments --paginate

# Review summaries (top-level review bodies)
gh api repos/{owner}/{repo}/pulls/{pr}/reviews --paginate

# General PR comments (discussion)
gh api repos/{owner}/{repo}/issues/{pr}/comments --paginate
```

**Capture for each comment:**
- `id` (for replying)
- `node_id` (for resolving threads via GraphQL)
- `body` (the comment text)
- `path` and `line` (for inline comments)
- `user.login` (who wrote it)
- `in_reply_to_id` (to reconstruct threads)

**Filtering rules:**
- Skip comments authored by the current user (your own replies)
- Skip administrative bot comments (e.g., "Auto-Review: Escalated", review badge summaries, approval status messages, CI/CD notifications)
- Do NOT skip bot comments that contain substantive code review feedback (e.g., from Devin, Greptile, CodeRabbit) — triage these the same as human comments
- The test is whether the comment contains actionable technical feedback about the code vs. bot housekeeping/status noise

**Thread reconstruction:**
- Group comments by thread using `in_reply_to_id` (comments sharing the same root form one thread)
- Triage the thread as a unit, not individual replies — read the full conversation to understand current state
- If the reviewer's concern was already resolved in the thread (e.g., they replied "looks good" or "never mind"), categorize as Already Addressed
- If the thread shows ongoing disagreement, triage based on the reviewer's latest position
- Only the root comment of a thread needs a reply and resolution — don't reply to intermediate messages

**Review summaries (top-level review bodies):**
- Reviews from `/reviews` contain a `body` (overall summary) and `state` (APPROVED, CHANGES_REQUESTED, COMMENTED)
- These have no file/line reference and no thread to resolve
- If the review body contains actionable feedback, triage it alongside inline comments
- If the review body is empty or just approval text, skip it
- To reply to review-level feedback, use a general PR comment:
  ```bash
  gh api repos/{owner}/{repo}/issues/{pr}/comments -f body="Response text"
  ```

## Step 3: Triage

For each comment, evaluate against the actual codebase:

1. Read the referenced file and line
2. Understand the reviewer's point
3. Check if it's technically valid for THIS codebase
4. Categorize:

| Category | Criteria |
|----------|----------|
| **Fix** | Valid feedback, should be addressed |
| **Push Back** | Technically wrong, missing context, YAGNI, or not applicable |
| **Already Addressed** | Fixed in a later commit, or thread is stale |

**Triage rules:**
- Verify against codebase reality before categorizing
- Check if suggestion breaks existing functionality
- Check if reviewer has full context
- Apply YAGNI: if reviewer suggests a feature, grep for actual usage
- If conflicts with user's prior architectural decisions → Push Back

## Step 4: Present the Triage

Show all comments in a numbered list grouped by category:

```
## PR #142 - Code Review Triage

### Fix (N)
1. **src/file.ts:45** - @reviewer: "Comment text"
   → Reasoning for why this should be fixed.

### Push Back (N)
2. **src/file.ts:89** - @reviewer: "Comment text"
   → Reasoning for why this should be pushed back on.

### Already Addressed (N)
3. **src/file.ts:30** - @reviewer: "Comment text"
   → Which commit addressed it.

---
Anything you want to change before I proceed?
```

**Wait for user approval.** User may:
- Approve as-is
- Move items between categories (e.g., "move 3 to push back")
- Adjust reasoning for push backs

## Step 5: Execute

After approval, process all items in one pass.

**Fix ordering:** When multiple fixes touch the same file, process them bottom-to-top (highest line number first). This prevents earlier fixes from shifting the line numbers of later fixes.

### Fix Items

For each:
1. Implement the fix
2. Run tests to verify no regressions
   - If the fix caused the failure: revise the fix until tests pass before proceeding
   - If pre-existing failure: note it as a caveat, continue with other fixes
   - Never reply "Fixed" to a thread if the fix introduced test failures
3. Reply in the comment thread:
   ```bash
   gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
     -f body="Fixed - brief description of what changed."
   ```
4. Resolve the thread:
   ```bash
   gh api graphql -f query='
     mutation { resolveReviewThread(input: { threadId: "NODE_ID" }) { thread { isResolved } } }
   '
   ```

### Push Back Items

For each:
1. Reply with technical reasoning:
   ```bash
   gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
     -f body="Technical reasoning for why this doesn't apply."
   ```
2. Do NOT resolve the thread (let the reviewer respond)

### Already Addressed Items

For each:
1. Reply noting which commit:
   ```bash
   gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
     -f body="Addressed in abc123."
   ```
2. Resolve the thread

## Step 6: Push

```bash
# Stage only the files you modified during fixes — NOT git add -A
git add <files changed during fix execution>
git commit -m "fix: address PR review feedback"
git push
```

Do not use `git add -A` or `git add .` — these can stage secrets, build artifacts, or unrelated changes. Track which files each fix touched and stage them by name.

Report summary: "N fixed, N pushed back, N already addressed. Pushed."

## Reply Tone

**Never:**
- "Great catch!" / "Thanks!" / "You're absolutely right!"
- Any performative or gratitude language

**Instead:**
- `"Fixed - added null check with early return."`
- `"Switched to crypto.timingSafeEqual."`
- `"This is a server-side flow - PKCE is for public clients. Not applicable here."`

Keep replies to one line where possible. Technical, factual, direct.

## Red Flags

**Never:**
- Fix comments without user approval of the triage
- Resolve "Push Back" threads (reviewer needs to respond)
- Use performative language in replies
- Skip testing after fixes

**Always:**
- Fetch ALL comments before triaging
- Include GraphQL node_id during fetch (needed for resolution)
- Wait for user approval before executing
- Test after each fix
- Push back when feedback is wrong (with reasoning)

## Quick Reference

| Step | Action |
|------|--------|
| 1 | Detect PR from branch or URL |
| 2 | Fetch all comments via `gh api` |
| 3 | Triage: Fix / Push Back / Already Addressed |
| 4 | Present numbered batch, wait for approval |
| 5 | Execute: fix, reply, resolve |
| 6 | Commit, push, report summary |

## Integration

**Triggered by:** User running `/address-review-comments`

**Pairs with:**
- **superpowers:receiving-code-review** - Principles for evaluating feedback (used during triage)

**Uses:**
- `gh` CLI for all GitHub interactions
- GraphQL API for thread resolution

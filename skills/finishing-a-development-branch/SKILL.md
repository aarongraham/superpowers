---
name: finishing-a-development-branch
description: Use when implementation is complete and all tests pass - verifies work and creates PR
---

# Finishing a Development Branch

## Overview

Verify tests pass, then create a PR. Simple.

**Announce at start:** "I'm using the finishing-a-development-branch skill to complete this work."

## The Process

### Step 1: Verify Tests

```bash
# Run project's test suite
npm test / cargo test / pytest / go test ./...
```

**If tests fail:**
```
Tests failing (<N> failures). Must fix before creating PR:

[Show failures]
```

Stop. Don't proceed until tests pass.

### Step 2: Ask About PR

```
Tests pass. Would you like me to create a PR for this?
```

Wait for confirmation.

### Step 3: Create PR

Detect platform and create PR:

```bash
# Check for GitHub
gh auth status 2>/dev/null && gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"

# Or GitLab
glab auth status 2>/dev/null && glab mr create --title "<title>" --description "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

Report the PR/MR URL when done.

## Quick Reference

| Step | Action |
|------|--------|
| 1 | Run tests |
| 2 | Ask about PR |
| 3 | Create PR with gh/glab |

## Red Flags

**Never:**
- Create PR with failing tests
- Force-push without explicit request
- Skip the confirmation question

## Integration

**Called by:**
- **subagent-driven-development** - After all tasks complete
- **executing-plans** - After all tasks complete

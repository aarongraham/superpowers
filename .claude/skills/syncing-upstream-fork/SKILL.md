---
name: syncing-upstream-fork
description: Use when updating the aarongraham/superpowers fork to a new version of obra/superpowers upstream
---

# Syncing Upstream Fork

## Overview

Update this fork to the latest obra/superpowers release using a selective merge approach. Infrastructure changes are taken wholesale; skill changes are merged individually, preserving fork opinions.

**Core principle:** Two commits — infrastructure first, then selective skill merges with documented decisions.

## Process

### Step 0: Fetch Upstream

```bash
git remote add upstream https://github.com/obra/superpowers.git 2>/dev/null || true
git fetch upstream --tags
```

Identify the latest tag and current fork version:
```bash
git tag -l 'v*' --sort=-v:refname | head -5
cat .claude-plugin/plugin.json | grep version
```

### Step 1: Assess What Changed

```bash
# Overall stats
git diff --stat <current-tag>..<latest-tag>

# Infrastructure changes
git diff --stat <current-tag>..<latest-tag> -- hooks/ .claude-plugin/ .codex/ .cursor-plugin/ tests/ .gitattributes .gitignore

# Skill changes
git diff --stat <current-tag>..<latest-tag> -- skills/

# New/deleted files
git diff --diff-filter=A --name-only <current-tag>..<latest-tag>
git diff --diff-filter=D --name-only <current-tag>..<latest-tag>

# Check for new upstream skills we don't have
diff <(git ls-tree --name-only <latest-tag> skills/ | sort) <(ls -1 skills/ | sort)
```

Present the rundown to the user before proceeding.

### Step 2: Create Branch

```bash
git checkout -b ag/update-to-<version>
```

### Step 3: Commit 1 — Infrastructure

Take all non-skill infrastructure changes directly from upstream:

```bash
git checkout <latest-tag> -- hooks/ .cursor-plugin/ .gitignore <new-files...>
git checkout <latest-tag> -- tests/
```

Handle deleted files:
```bash
git rm <deleted-files>
```

Bump versions in `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`.

Commit with: `chore: update infrastructure files to match upstream v<version>`

Include a bullet list in the commit body of what changed.

### Step 4: Commit 2 — Selective Skill Merges

For each changed skill, diff upstream against the fork and decide what to take.

**Skills with no fork opinions** — take wholesale:
```bash
git checkout <latest-tag> -- skills/<skill-name>/
```

**Skills with fork opinions** — merge selectively. Read both versions, write the merged result preserving fork opinions (see list below).

**New upstream files** (supporting docs, prompt templates, scripts) — take wholesale unless they conflict with fork opinions.

Commit with: `feat: selectively merge upstream v<version> skill changes`

The commit body MUST document for each skill:
- What was taken and why
- What was intentionally NOT taken and why

### Step 5: Optional Fork-Specific Improvements

If there are new fork-specific improvements worth making on top of the sync, add as a separate commit with a `feat:` prefix.

### Step 6: Push and Create PR

```bash
git push -u origin ag/update-to-<version>
```

Note: `gh pr create` may fail due to enterprise auth — provide the GitHub URL for manual PR creation if needed.

## Fork Opinions to Preserve

These are intentional divergences from upstream. Do NOT overwrite with upstream changes:

| Skill | Opinion | Rationale |
|-------|---------|-----------|
| `brainstorming` | Do NOT auto-commit design docs | Design docs may be gitignored |
| `brainstorming` | Keep `docs/plans/` path | Fork uses flat path, not `docs/superpowers/specs/` |
| `writing-plans` | Keep direct-to-subagent execution handoff | No execution choice offered; proceeds immediately |
| `writing-plans` | Keep `docs/plans/` path | Fork uses flat path, not `docs/superpowers/plans/` |
| `subagent-driven-development` | Do NOT adopt worktree requirements | Fork doesn't require worktrees |
| `subagent-driven-development` | Keep `docs/plans/` path | Consistent with writing-plans |
| `finishing-a-development-branch` | Keep simple verify+PR flow | Do NOT adopt rewrites |
| `using-git-worktrees` | Auto-create `.worktrees/` as default | Do NOT change default behavior |
| `systematic-debugging` | Keep softened enforcement language | "Related" not "REQUIRED", "discuss with human" not "auto-escalate" |
| `address-review-comments` | Fork-only skill — preserve unchanged | Upstream doesn't have it |
| `planning-agent-teams` | Fork-only skill — preserve unchanged | Upstream doesn't have it |

**If upstream changes align with a fork opinion** (e.g., upstream removed batch execution in v5.0.6, matching our preference), note this in the commit message but still verify the final state preserves the opinion.

## Verification

Run after all commits, before pushing:

```bash
# All skill frontmatter valid
for f in skills/*/SKILL.md; do
  name=$(head -5 "$f" | grep "^name:"); desc=$(head -5 "$f" | grep "^description:");
  [ -z "$name" ] || [ -z "$desc" ] && echo "BROKEN: $f" || echo "OK: $f"
done

# Hook scripts executable
ls -la hooks/session-start hooks/run-hook.cmd

# Fork opinions intact
grep "docs/plans" skills/brainstorming/SKILL.md
grep "docs/plans" skills/writing-plans/SKILL.md
grep -c "Commit the design" skills/brainstorming/SKILL.md  # should be 0
grep "Discuss with" skills/systematic-debugging/SKILL.md
ls skills/address-review-comments/SKILL.md skills/planning-agent-teams/SKILL.md

# No missing upstream skills
diff <(git ls-tree --name-only <latest-tag> skills/ | sort) <(ls -1 skills/ | sort)
```

## Prior Syncs

| PR | From → To | Notes |
|----|-----------|-------|
| #4 | v4.3.0 → v4.3.1 | Added Sonnet sub-agent gate (later removed in v5.0.6 sync) |
| #5 | v4.3.1 → v5.0.6 | Upstream aligned with fork on execute-all; added visual companion, self-review, model selection |

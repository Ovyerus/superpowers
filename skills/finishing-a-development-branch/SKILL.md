---
name: finishing-a-development-branch
description: Use when implementation is complete, all tests pass, and you need to decide how to integrate the work - guides completion of development work by presenting structured options for merge, PR, or cleanup
---

# Finishing a Development Branch

## Overview

Guide completion of development work by presenting clear options and handling chosen workflow.

**Core principle:** Verify tests → Present options → Execute choice → Clean up.

## VCS Detection

```bash
[ -d .jj ] && echo "USE_JJ=true" || echo "USE_JJ=false"
```

All steps below show `# git:` and `# jj:` variants.

**Announce at start:** "I'm using the finishing-a-development-branch skill to complete this work."

## The Process

### Step 1: Verify Tests

**Before presenting options, verify tests pass:**

```bash
# Run project's test suite
npm test / cargo test / pytest / go test ./...
```

**If tests fail:**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

Stop. Don't proceed to Step 2.

**If tests pass:** Continue to Step 2.

### Step 2: Determine Base Branch

```bash
# git:
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null

# jj:
jj log -r 'lca(@, main)' --no-graph -T 'commit_id' 2>/dev/null || \
jj log -r 'lca(@, master)' --no-graph -T 'commit_id' 2>/dev/null
```

> **jj note:** `lca()` (lowest common ancestor) is the revset equivalent of `git merge-base`.

Or ask: "This branch split from main - is that correct?"

### Step 3: Present Options

Present exactly these 4 options:

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**Don't add explanation** - keep options concise.

### Step 4: Execute Choice

#### Option 1: Merge Locally

```bash
# git:
git checkout <base-branch>
git pull
git merge <feature-branch>
<test command>
git branch -d <feature-branch>

# jj:
# Fetch latest from remote
jj git fetch
# Create merge commit with two parents (base bookmark + feature bookmark)
jj new <base-bookmark> <feature-bookmark>
# Move the base bookmark forward to this merge commit
jj bookmark move <base-bookmark> -r @
# Verify tests
<test command>
# Delete the feature bookmark (commits remain in history)
jj bookmark delete <feature-bookmark>
```

> **jj merge semantics:** jj has no `checkout` + `merge` sequence. `jj new <rev1> <rev2>` creates a new change with two parents. Then `jj bookmark move` advances the base bookmark. `jj git fetch` replaces `git pull` — unlike `git pull`, it only fetches without auto-integrating.

Then: Cleanup worktree (Step 5)

#### Option 2: Push and Create PR

```bash
# git:
git push -u origin <feature-branch>

# jj:
jj git push --bookmark <feature-bookmark>
```

> **jj push:** No `-u` flag needed — jj tracks the remote automatically.

```bash
# Create PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

Then: Cleanup worktree (Step 5)

#### Option 3: Keep As-Is

Report: "Keeping branch <name>. Worktree preserved at <path>."

**Don't cleanup worktree.**

#### Option 4: Discard

**Confirm first:**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

Wait for exact confirmation.

If confirmed:
```bash
# git:
git checkout <base-branch>
git branch -D <feature-branch>

# jj:
jj bookmark delete <feature-bookmark>
# jj workspaces are independent — no checkout needed.
# Abandoned commits are garbage collected automatically.
```

Then: Cleanup worktree (Step 5)

### Step 5: Cleanup Worktree

**For Options 1, 2, 4:**

Check if in worktree:
```bash
# git:
git worktree list | grep $(git branch --show-current)

# jj:
jj workspace list
# (workspace names are directory basenames; find the one matching this feature)
```

If yes:
```bash
# git:
git worktree remove <worktree-path>

# jj:
jj workspace forget <workspace-name>
rm -rf <worktree-path>
# Note: jj workspace forget takes the NAME (from jj workspace list), not the path.
#       It does NOT delete the directory — rm -rf is required.
```

**For Option 3:** Keep worktree.

## Quick Reference

| Option | Merge | Push | Keep Worktree | Cleanup Branch |
|--------|-------|------|---------------|----------------|
| 1. Merge locally | ✓ | - | - | ✓ |
| 2. Create PR | - | ✓ | ✓ | - |
| 3. Keep as-is | - | - | ✓ | - |
| 4. Discard | - | - | - | ✓ (force) |

## Common Mistakes

**Skipping test verification**
- **Problem:** Merge broken code, create failing PR
- **Fix:** Always verify tests before offering options

**Open-ended questions**
- **Problem:** "What should I do next?" → ambiguous
- **Fix:** Present exactly 4 structured options

**Automatic worktree cleanup**
- **Problem:** Remove worktree when might need it (Option 2, 3)
- **Fix:** Only cleanup for Options 1 and 4

**No confirmation for discard**
- **Problem:** Accidentally delete work
- **Fix:** Require typed "discard" confirmation

## Red Flags

**Never:**
- Proceed with failing tests
- Merge without verifying tests on result
- Delete work without confirmation
- Force-push without explicit request

**Always:**
- Verify tests before offering options
- Present exactly 4 options
- Get typed confirmation for Option 4
- Clean up worktree for Options 1 & 4 only

## Integration

**Called by:**
- **subagent-driven-development** (Step 7) - After all tasks complete
- **executing-plans** (Step 5) - After all batches complete

**Pairs with:**
- **using-git-worktrees** - Cleans up worktree created by that skill

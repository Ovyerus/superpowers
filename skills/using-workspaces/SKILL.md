---
name: using-workspaces
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - creates isolated jj workspaces with smart directory selection and safety verification
---

# Using jj Workspaces

## Overview

jj workspaces create isolated working copies sharing the same repository, allowing work on multiple bookmarks simultaneously without switching.

**Core principle:** Systematic directory selection + safety verification = reliable isolation.

**Announce at start:** "I'm using the using-workspaces skill to set up an isolated workspace."

## Directory Selection Process

Follow this priority order:

### 1. Check Existing Directories

```bash
ls -d .worktrees 2>/dev/null     # Preferred (hidden)
ls -d worktrees 2>/dev/null      # Alternative
```

**If found:** Use that directory. If both exist, `.worktrees` wins.

### 2. Check CLAUDE.md

```bash
grep -i "worktree.*director" CLAUDE.md 2>/dev/null
```

**If preference specified:** Use it without asking.

### 3. Ask User

If no directory exists and no CLAUDE.md preference:

```
No workspace directory found. Where should I create workspaces?

1. .worktrees/ (project-local, hidden)
2. ~/.config/superpowers/worktrees/<project-name>/ (global location)

Which would you prefer?
```

## Safety Verification

### For Project-Local Directories (.worktrees or worktrees)

**MUST verify directory is ignored before creating workspace:**

```bash
# jj uses .gitignore — this command works in all jj repos
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**If NOT ignored:**

Per Jesse's rule "Fix broken things immediately":

1. Add appropriate line to .gitignore
2. Run `jj file untrack <path>` if needed
3. Commit: `jj commit -m "chore: ignore workspace directory"`
4. Proceed with workspace creation

**Why critical:** Prevents accidentally tracking workspace contents.

### For Global Directory (~/.config/superpowers/worktrees)

No .gitignore verification needed - outside project entirely.

## Creation Steps

### 1. Detect Project Name

```bash
project=$(basename "$(jj root)")
```

### 2. Create Workspace

```bash
# Determine full path
case $LOCATION in
  .worktrees|worktrees)
    path="$LOCATION/$BOOKMARK_NAME"
    ;;
  ~/.config/superpowers/worktrees/*)
    path="~/.config/superpowers/worktrees/$project/$BOOKMARK_NAME"
    ;;
esac

# Step 1: Create workspace (does NOT auto-create a bookmark)
jj workspace add "$path"
cd "$path"
# Step 2: Create bookmark so this workspace has a named branch
jj bookmark create "$BOOKMARK_NAME" -r @
```

> **jj workspace semantics:** `jj workspace add` creates the workspace at the current `@` revision. It does NOT create a bookmark. You must run `jj bookmark create <name> -r @` in the new workspace to name the branch. `$BOOKMARK_NAME` is the intended feature name.

### 3. Run Project Setup

Auto-detect and run appropriate setup:

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

### 4. Verify Clean Baseline

Run tests to ensure workspace starts clean:

```bash
# Examples - use project-appropriate command
npm test
cargo test
pytest
go test ./...
```

**If tests fail:** Report failures, ask whether to proceed or investigate.

**If tests pass:** Report ready.

### 5. Report Location

```
Workspace ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## Quick Reference

| Situation                  | Action                                                             |
| -------------------------- | ------------------------------------------------------------------ |
| `.worktrees/` exists       | Use it (verify ignored)                                            |
| `worktrees/` exists        | Use it (verify ignored)                                            |
| Both exist                 | Use `.worktrees/`                                                  |
| Neither exists             | Check CLAUDE.md → Ask user                                         |
| Directory not ignored      | Add to .gitignore + `jj file untrack` + `jj commit`               |
| Detect project root        | `jj root`                                                          |
| Create workspace           | `jj workspace add "$path"` then `jj bookmark create "$NAME" -r @` |
| Check ignored              | `git check-ignore -q <dir>` (jj uses .gitignore format)            |
| Tests fail during baseline | Report failures + ask                                              |
| No package.json/Cargo.toml | Skip dependency install                                            |

## Common Mistakes

### Skipping ignore verification

- **Problem:** Workspace contents get tracked
- **Fix:** Always use `git check-ignore` before creating project-local workspace

### Assuming directory location

- **Problem:** Creates inconsistency, violates project conventions
- **Fix:** Follow priority: existing > CLAUDE.md > ask

### Proceeding with failing tests

- **Problem:** Can't distinguish new bugs from pre-existing issues
- **Fix:** Report failures, get explicit permission to proceed

### Hardcoding setup commands

- **Problem:** Breaks on projects using different tools
- **Fix:** Auto-detect from project files (package.json, etc.)

## Example Workflow

```
You: I'm using the using-workspaces skill to set up an isolated workspace.

[Check .worktrees/ - exists]
[Verify ignored - git check-ignore confirms .worktrees/ is ignored]
[jj workspace add .worktrees/auth]
[cd .worktrees/auth]
[jj bookmark create feature/auth -r @]
[Run npm install]
[Run npm test - 47 passing]

Workspace ready at /Users/ovy/myproject/.worktrees/auth
Tests passing (47 tests, 0 failures)
Ready to implement auth feature
```

## Red Flags

**Never:**

- Create workspace without verifying it's ignored (project-local)
- Skip baseline test verification
- Proceed with failing tests without asking
- Assume directory location when ambiguous
- Skip CLAUDE.md check

**Always:**

- Follow directory priority: existing > CLAUDE.md > ask
- Verify directory is ignored for project-local
- Auto-detect and run project setup
- Verify clean test baseline

## Integration

**Called by:**

- **brainstorming** (Phase 4) - REQUIRED when design is approved and implementation follows
- **subagent-driven-development** - REQUIRED before executing any tasks
- **executing-plans** - REQUIRED before executing any tasks
- Any skill needing isolated workspace

**Pairs with:**

- **finishing-a-development-branch** - REQUIRED for cleanup after work complete

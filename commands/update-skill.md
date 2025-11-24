---
name: update-skill
description: Update locally installed skills from their GitHub repositories
arguments:
  - name: target
    description: "Repository name to update, or --all to update all repos"
    required: false
---

# Update Locally Installed Skills

You are updating Claude Code skills that were installed via `/install-skill`.

## Parse the Input

Parse the target argument: `$ARGUMENTS.target`

- If empty or not provided: List installed repos and ask which to update
- If `--all`: Update all repos in registry
- Otherwise: Update the specified repo

## Update Process

### Step 1: Read registry

Read `.claude/local-plugins.json`.

If file doesn't exist:
```
No locally installed skills found.

Use /install-skill to install skills from GitHub.
```

### Step 2: Identify repos to update

If `--all`:
- Get all repo names from registry

If specific repo:
- Verify repo exists in registry
- If not found, list available repos

If no target specified:
- List all repos with their installed skills
- Ask user which to update

### Step 3: Update each repo

For each repo to update:

```bash
cd .claude/plugins/local/<repo>
git fetch origin
git reset --hard origin/<branch>
```

Capture the old and new commit SHA:
```bash
# Before
OLD_SHA=$(git rev-parse HEAD)

# After fetch and reset
NEW_SHA=$(git rev-parse HEAD)
```

### Step 4: Verify symlinks

For each installed skill from this repo:
1. Check if symlink still exists in `.claude/skills/`
2. Check if symlink target still exists (skill wasn't removed from repo)
3. If target missing, warn user and offer to remove symlink

### Step 5: Check for new skills

List skills in updated repo:
```bash
ls .claude/plugins/local/<repo>/skills/
```

Compare with `installedSkills` in registry.

If new skills available:
```
New skills available in <repo>:
  - new-skill-a
  - new-skill-b

Install with: /install-skill <owner>/<repo>:new-skill-a
```

### Step 6: Update registry

Update the repo entry in `.claude/local-plugins.json`:
- Update `gitCommitSha` to new SHA
- Keep `installedSkills` unchanged (only /install-skill modifies this)

### Step 7: Report results

**No changes:**
```
Updating <repo>...
└── Already up to date (sha: abc123)
```

**Updated:**
```
Updating <repo>...
├── Previous: abc123
├── Updated to: def456
├── Changes: 3 commits
└── Symlinks verified: 2 skills OK

Skills updated successfully.
```

**With warnings:**
```
Updating <repo>...
├── Previous: abc123
├── Updated to: def456
├── Warning: Skill "old-skill" was removed from repo
│   └── Symlink .claude/skills/old-skill is now broken
│   └── Run: /remove-skill old-skill
└── 1 of 2 skills OK

Update completed with warnings.
```

## Error Handling

**Repo directory missing:**
```
Error: Repository directory not found: .claude/plugins/local/<repo>

The repository may have been manually deleted.
Run /install-skill to reinstall, or /remove-skill to clean up registry.
```

**Network error:**
```
Error: Failed to fetch updates for <repo>

Check your network connection and try again.
```

**Merge conflicts (shouldn't happen with --hard reset):**
```
Error: Failed to update <repo>

Try removing and reinstalling:
  /remove-skill <skill-name> --purge
  /install-skill <owner>/<repo>:<skill-name>
```

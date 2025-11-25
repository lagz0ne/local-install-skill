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

- If empty or not provided: List installed skills and ask which to update
- If `--all`: Update all updateable skills
- Otherwise: Update the specified skill or repo

## Update Process

### Step 1: Read registry

Read `.claude/local-plugins.yaml`.

**Migration check:** If `.claude/local-plugins.json` exists but `.yaml` doesn't, run migration first.

If no registry found:
```
No locally installed skills found.

Use /install-skill to install skills from GitHub.
```

### Step 2: Identify what to update

**If `--all`:**
- Get all skills from registry
- Group by mode for appropriate update method

**If specific target:**
- Check if it's a skill name in `skills` section
- Check if it's a repo name in `submodules` section
- If not found, list available options

**If no target:**
- List all skills with their mode and source
- Ask user which to update

### Step 3: Update based on mode

#### For submodule-based skills:

Update the entire submodule (updates all skills from that repo):

```bash
git -C .claude/submodules/<repo> fetch origin
git -C .claude/submodules/<repo> pull origin <branch>
```

Capture old and new SHA:
```bash
OLD_SHA=$(git -C .claude/submodules/<repo> rev-parse HEAD)
# After pull
NEW_SHA=$(git -C .claude/submodules/<repo> rev-parse HEAD)
```

Update all skills from this repo in registry with new `commitSha`.

#### For copy-based skills:

Re-download and overwrite:

```bash
TEMP_DIR=$(mktemp -d)
git clone --depth=1 --branch <branch> https://github.com/<owner>/<repo>.git "$TEMP_DIR"
NEW_SHA=$(git -C "$TEMP_DIR" rev-parse HEAD)
```

Compare with stored `commitSha`. If same:
```
Skill "<name>" is already up to date (sha: <sha>)
```

If different:
```bash
rm -rf ".claude/skills/<skill-name>"
cp -r "$TEMP_DIR/<skillPath>" ".claude/skills/<skill-name>"
rm -rf "$TEMP_DIR"
```

Update `commitSha` in registry.

### Step 4: Verify integrity

**For submodule skills:**
- Check symlinks still point to valid targets
- If skill was removed from upstream repo, warn user

**For copy skills:**
- Verify files were copied successfully

### Step 5: Update registry

Write updated `.claude/local-plugins.yaml` with new `commitSha` values.

### Step 6: Report results

**No changes:**
```
Updating <skill-name>...
└── Already up to date (sha: abc123)
```

**Updated (submodule):**
```
Updating submodule <repo>...
├── Previous: abc123
├── Updated to: def456
├── Skills updated: brainstorming, tdd
└── Symlinks verified: OK

Skills updated successfully.
```

**Updated (copy):**
```
Updating skill <name>...
├── Previous: abc123
├── Updated to: def456
├── Files replaced in: .claude/skills/<name>
└── Status: Updated successfully
```

## Error Handling

**Submodule directory missing:**
```
Error: Submodule not found: .claude/submodules/<repo>

Run `git submodule update --init` or reinstall with /install-skill.
```

**Network error:**
```
Error: Failed to fetch updates

Check your network connection and try again.
```

**Skill not in registry:**
```
Error: Skill "<name>" not found in registry.

Installed skills:
  - skill-a (copy mode)
  - skill-b (submodule mode)
```

---
name: remove-skill
description: Remove a locally installed skill
arguments:
  - name: skill_name
    description: "Name of the skill to remove"
    required: true
  - name: purge
    description: "Also remove the cloned repository if no other skills are installed from it"
    required: false
---

# Remove Locally Installed Skill

You are removing a Claude Code skill that was installed via `/install-skill`.

## Parse the Input

- `skill_name`: `$ARGUMENTS.skill_name` - The skill to remove
- `purge`: Check if `--purge` flag is present in arguments

## Removal Process

### Step 1: Verify skill exists

Check if `.claude/skills/<skill_name>` exists.

If not found:
```
Error: Skill "<skill_name>" not found in .claude/skills/

Installed skills:
  - skill-a (from repo-x)
  - skill-b (from repo-y)
```

List installed skills by reading symlinks and registry.

### Step 2: Check if it's a symlink

```bash
if [ -L ".claude/skills/<skill_name>" ]; then
    # It's a symlink (installed by us)
    TARGET=$(readlink ".claude/skills/<skill_name>")
else
    # It's a regular directory (not installed by us)
fi
```

If not a symlink:
```
Warning: .claude/skills/<skill_name> is not a symlink.
This skill was not installed via /install-skill.

Remove anyway? This will delete the directory and its contents.
```

Ask for confirmation before proceeding with non-symlink removal.

### Step 3: Identify source repository

Parse the symlink target to find the repo:
```
../plugins/local/<repo>/skills/<skill_name>
```

Extract `<repo>` from the path.

### Step 4: Remove symlink

```bash
rm .claude/skills/<skill_name>
```

### Step 5: Update registry

Read `.claude/local-plugins.json`.

Remove skill from repo's `installedSkills` array:
```json
{
  "repos": {
    "<repo>": {
      "installedSkills": ["skill-a", "skill-b"]  // Remove the uninstalled one
    }
  }
}
```

### Step 6: Handle --purge flag

If `--purge` flag is set:

Check if repo has other installed skills:
```javascript
if (repo.installedSkills.length === 0) {
    // No other skills, safe to remove repo
}
```

If no other skills installed from this repo:
```bash
rm -rf .claude/plugins/local/<repo>
```

Remove repo entry from registry entirely.

If other skills still installed:
```
Note: Repository <repo> still has other skills installed:
  - other-skill-a
  - other-skill-b

Repository not removed. Use /remove-skill for each, or manually delete.
```

### Step 7: Clean up empty directories

```bash
# Remove empty skills directory if no skills left
rmdir .claude/skills 2>/dev/null || true

# Remove empty plugins/local directory if no repos left
rmdir .claude/plugins/local 2>/dev/null || true
rmdir .claude/plugins 2>/dev/null || true
```

### Step 8: Report success

**Basic removal:**
```
Removing skill "<skill_name>"...
├── Removed symlink: .claude/skills/<skill_name>
├── Updated registry
└── Repository <repo> retained (other skills installed)

Skill "<skill_name>" has been removed.
```

**With purge:**
```
Removing skill "<skill_name>"...
├── Removed symlink: .claude/skills/<skill_name>
├── Removed repository: .claude/plugins/local/<repo>
└── Cleaned up registry

Skill "<skill_name>" and its repository have been removed.
```

## List Installed Skills

If user runs `/remove-skill` without arguments or with `--list`:

```
Locally installed skills:

From superpowers-marketplace (github.com/obra/superpowers-marketplace):
  - brainstorming
  - tdd
  - systematic-debugging

From my-team-skills (github.com/myorg/team-skills):
  - code-review
  - deployment

Use: /remove-skill <skill-name> [--purge]
```

## Error Handling

**Skill not in registry but symlink exists:**
```
Warning: Skill "<skill_name>" found but not in registry.

This may be a manually created symlink or from an older installation.
Remove the symlink anyway? (y/n)
```

**Permission error:**
```
Error: Permission denied removing .claude/skills/<skill_name>

Check file permissions and try again.
```

**Registry corrupted:**
```
Warning: Could not parse .claude/local-plugins.json

Proceeding with symlink removal only.
Consider manually fixing or deleting the registry file.
```

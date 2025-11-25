---
name: remove-skill
description: Remove a locally installed skill
arguments:
  - name: skill_name
    description: "Name of the skill to remove"
    required: true
---

# Remove Locally Installed Skill

You are removing a Claude Code skill that was installed via `/install-skill`.

## Parse the Input

- `skill_name`: `$ARGUMENTS.skill_name` - The skill to remove

## Removal Process

### Step 1: Read registry and find skill

Read `.claude/local-plugins.yaml`.

Look up `skills.<skill_name>`. If not found:
```
Error: Skill "<skill_name>" not found in registry.

Installed skills:
  - skill-a (copy)
  - skill-b (submodule)

Use the exact skill name from the list above.
```

Extract the skill's `mode` for appropriate removal.

### Step 2: Remove skill files/symlink

#### If mode is "copy":

```bash
rm -rf ".claude/skills/<skill_name>"
```

#### If mode is "submodule":

```bash
rm ".claude/skills/<skill_name>"  # Remove symlink only
```

### Step 3: Update registry - skills section

Remove the skill entry from `skills` section.

### Step 4: Handle submodule cleanup (submodule mode only)

**Skip if mode is "copy".**

Check if other skills still use this submodule:
1. Read `submodules.<repo>.skills` list
2. Remove `<skill_name>` from the list
3. If list is now empty, remove the submodule:

```bash
git submodule deinit .claude/submodules/<repo>
git rm .claude/submodules/<repo>
rm -rf .git/modules/.claude/submodules/<repo>
```

Remove the repo entry from `submodules` section.

If other skills still use the submodule:
```
Note: Submodule .claude/submodules/<repo> retained.
Other skills still using it: skill-x, skill-y
```

### Step 5: Clean up empty directories

```bash
# Remove empty skills directory if no skills left
rmdir .claude/skills 2>/dev/null || true

# Remove empty submodules directory if no submodules left
rmdir .claude/submodules 2>/dev/null || true
```

### Step 6: Write updated registry

Write `.claude/local-plugins.yaml`.

If no skills remain, consider removing the file:
```bash
if [ -z "$(grep -v '^version:' .claude/local-plugins.yaml | grep -v '^skills: {}' | grep -v '^submodules: {}')" ]; then
    rm .claude/local-plugins.yaml
fi
```

### Step 7: Report success

**Copy mode removal:**
```
Removing skill "<skill_name>"...
├── Mode: Copy
├── Removed: .claude/skills/<skill_name>/
└── Registry updated

Skill "<skill_name>" has been removed.
```

**Submodule mode removal (submodule kept):**
```
Removing skill "<skill_name>"...
├── Mode: Submodule
├── Removed symlink: .claude/skills/<skill_name>
├── Submodule retained: .claude/submodules/<repo>
│   └── Other skills: skill-x, skill-y
└── Registry updated

Skill "<skill_name>" has been removed.
```

**Submodule mode removal (submodule removed):**
```
Removing skill "<skill_name>"...
├── Mode: Submodule
├── Removed symlink: .claude/skills/<skill_name>
├── Removed submodule: .claude/submodules/<repo>
└── Registry updated

Skill "<skill_name>" and its submodule have been removed.
```

## List Installed Skills

If user runs `/remove-skill` without arguments or with `--list`:

```
Locally installed skills:

Copy mode:
  - my-custom-skill (from github.com/user/repo)

Submodule mode:
  - brainstorming (from github.com/obra/superpowers)
  - tdd (from github.com/obra/superpowers)

Use: /remove-skill <skill-name>
```

## Error Handling

**Skill files missing but in registry:**
```
Warning: Skill "<skill_name>" in registry but files not found.
Cleaning up registry entry...

Done. Registry cleaned.
```

**Permission error:**
```
Error: Permission denied removing .claude/skills/<skill_name>

Check file permissions and try again.
```

**Submodule removal fails:**
```
Error: Failed to remove submodule .claude/submodules/<repo>

Manual cleanup may be required:
  git submodule deinit .claude/submodules/<repo>
  git rm .claude/submodules/<repo>
  rm -rf .git/modules/.claude/submodules/<repo>
```

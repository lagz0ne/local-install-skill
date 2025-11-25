# Skill Installation Modes - Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add copy and submodule installation modes to local-install-skill plugin, with interactive mode selection and YAML registry.

**Architecture:** Modify three command files (install, update, remove) to support two installation modes. Copy mode downloads and copies only skill files. Submodule mode uses git submodules with symlinks. Registry migrates from JSON to YAML with v2 schema.

**Tech Stack:** Bash (git commands), Claude Code slash commands (markdown), YAML registry format.

---

## Task 1: Update install-skill.md - Parse New Flags

**Files:**
- Modify: `/home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/install-skill.md:1-30`

**Step 1: Update frontmatter with new arguments**

Replace lines 1-8 with:

```markdown
---
name: install-skill
description: Install a skill from GitHub to project scope (.claude/skills/)
arguments:
  - name: source
    description: "GitHub repo with optional branch and skill path: user/repo, user/repo:skill-name, user/repo#branch:skill-name, or user/repo --all"
    required: true
  - name: mode
    description: "Installation mode: --copy (files only) or --submodule (git submodule, default)"
    required: false
---
```

**Step 2: Verify the change**

Run: `head -12 /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/install-skill.md`
Expected: Updated frontmatter with `mode` argument.

**Step 3: Commit**

```bash
cd /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill
git add commands/install-skill.md
git commit -m "feat(install): add mode argument to frontmatter"
```

---

## Task 2: Update install-skill.md - Add Mode Parsing Section

**Files:**
- Modify: `/home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/install-skill.md:16-30`

**Step 1: Add mode parsing after existing parsing section**

After the "Parse the Input" section (around line 30), add:

```markdown
## Determine Installation Mode

Check for mode flags in `$ARGUMENTS.mode` or within `$ARGUMENTS.source`:

**Flag detection:**
- If `--copy` present: `mode = "copy"`
- If `--submodule` present: `mode = "submodule"`
- If neither: Prompt user to choose

**Interactive prompt (when no flag):**

Use AskUserQuestion with these options:
- **Submodule (default)** - Full repo as git submodule, easy updates via `git submodule update`
- **Copy** - Only skill files copied, minimal footprint, manual re-download to update

Store the selected mode for use in later steps.
```

**Step 2: Verify the change**

Run: `grep -A 15 "Determine Installation Mode" /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/install-skill.md`
Expected: New section with mode detection logic.

**Step 3: Commit**

```bash
cd /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill
git add commands/install-skill.md
git commit -m "feat(install): add mode parsing and interactive prompt"
```

---

## Task 3: Update install-skill.md - Add Copy Mode Installation Flow

**Files:**
- Modify: `/home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/install-skill.md`

**Step 1: Replace Step 2 (clone check) and Step 3 (clone) with mode-aware logic**

Find the section starting with "### Step 2: Check if repo already cloned" and replace through "### Step 3: Clone the repository" with:

```markdown
### Step 2: Execute Installation Based on Mode

#### If mode is "copy":

**2a. Clone to temporary directory:**
```bash
TEMP_DIR=$(mktemp -d)
git clone --depth=1 https://github.com/<owner>/<repo>.git "$TEMP_DIR"
```

If specific branch:
```bash
git clone --depth=1 --branch <branch> https://github.com/<owner>/<repo>.git "$TEMP_DIR"
```

**2b. Get commit SHA before cleanup:**
```bash
COMMIT_SHA=$(git -C "$TEMP_DIR" rev-parse HEAD)
```

**2c. Copy skill directory:**
```bash
mkdir -p .claude/skills
cp -r "$TEMP_DIR/<skillPath>" ".claude/skills/<skill-name>"
```

**2d. Clean up temp directory:**
```bash
rm -rf "$TEMP_DIR"
```

#### If mode is "submodule":

**2a. Check if this is a git repository:**
```bash
git rev-parse --git-dir > /dev/null 2>&1
```

If not a git repo, error:
```
Error: Submodule mode requires a git repository.

Either initialize git first (git init) or use --copy mode.
```

**2b. Check if submodule already exists:**
```bash
if [ -d ".claude/submodules/<repo>" ]; then
    # Verify it's the same source
    EXISTING_URL=$(git config --file .gitmodules submodule..claude/submodules/<repo>.url)
fi
```

**2c. Add submodule if not exists:**
```bash
mkdir -p .claude/submodules
git submodule add https://github.com/<owner>/<repo>.git .claude/submodules/<repo>
```

If specific branch:
```bash
git submodule add -b <branch> https://github.com/<owner>/<repo>.git .claude/submodules/<repo>
```

**2d. Get commit SHA:**
```bash
COMMIT_SHA=$(git -C ".claude/submodules/<repo>" rev-parse HEAD)
```
```

**Step 2: Verify the change**

Run: `grep -A 5 "If mode is" /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/install-skill.md`
Expected: Both "copy" and "submodule" mode sections visible.

**Step 3: Commit**

```bash
cd /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill
git add commands/install-skill.md
git commit -m "feat(install): add copy and submodule installation flows"
```

---

## Task 4: Update install-skill.md - Update Symlink Creation for Modes

**Files:**
- Modify: `/home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/install-skill.md`

**Step 1: Replace Step 6 (symlink creation) with mode-aware logic**

Find "### Step 6: Create symlinks" and replace with:

```markdown
### Step 6: Create symlinks (submodule mode only)

**Skip this step if mode is "copy"** - files are already in `.claude/skills/<name>/`.

For submodule mode, create symlink:

Check if `.claude/skills/<skill-name>` already exists:
- If symlink pointing to same target: skip, already installed
- If symlink pointing elsewhere or regular directory: ask user to overwrite or skip

Create relative symlink:
```bash
mkdir -p .claude/skills
ln -s ../submodules/<repo>/<skillPath> .claude/skills/<skill-name>
```

For example, if skill is at `skills/brainstorming/`:
```bash
ln -s ../submodules/superpowers/skills/brainstorming .claude/skills/brainstorming
```
```

**Step 2: Verify the change**

Run: `grep -A 10 "Step 6: Create symlinks" /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/install-skill.md`
Expected: Mode-aware symlink logic.

**Step 3: Commit**

```bash
cd /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill
git add commands/install-skill.md
git commit -m "feat(install): mode-aware symlink creation"
```

---

## Task 5: Update install-skill.md - Update Registry to YAML v2

**Files:**
- Modify: `/home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/install-skill.md`

**Step 1: Replace Step 7 (registry update) with YAML v2 format**

Find "### Step 7: Update registry" and replace with:

```markdown
### Step 7: Update registry

Read or create `.claude/local-plugins.yaml`:

```yaml
version: 2
skills: {}
submodules: {}
```

**Migration:** If `.claude/local-plugins.json` exists (v1), migrate it first:
1. Read JSON content
2. Convert each repo's skills to v2 format with `mode: legacy`
3. Write to `.claude/local-plugins.yaml`
4. Delete `.claude/local-plugins.json`

**Add skill entry:**

```yaml
skills:
  <skill-name>:
    mode: <copy|submodule>
    source: github.com/<owner>/<repo>
    repo: <repo>
    branch: <branch>
    skillPath: <path/to/skill>
    installedAt: <ISO timestamp>
    commitSha: <sha>
```

**If submodule mode, also update submodules section:**

```yaml
submodules:
  <repo>:
    source: github.com/<owner>/<repo>
    path: .claude/submodules/<repo>
    skills:
      - <skill-name>
```

If repo already in submodules, just append skill name to the `skills` list.

**Write YAML:** Use proper YAML formatting with 2-space indentation.
```

**Step 2: Verify the change**

Run: `grep -A 20 "Step 7: Update registry" /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/install-skill.md`
Expected: YAML v2 format instructions.

**Step 3: Commit**

```bash
cd /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill
git add commands/install-skill.md
git commit -m "feat(install): migrate registry to YAML v2 format"
```

---

## Task 6: Update install-skill.md - Update .gitignore Handling

**Files:**
- Modify: `/home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/install-skill.md`

**Step 1: Replace Step 8 (.gitignore) with mode-aware logic**

Find "### Step 8: Update .gitignore" and replace with:

```markdown
### Step 8: Update .gitignore (copy mode only)

**Skip this step for submodule mode** - submodules are tracked by git naturally.

For copy mode, the skill files in `.claude/skills/<name>/` will be committed directly. No .gitignore changes needed.

**Note:** The old `.claude/plugins/local/` pattern can be removed from .gitignore if present, as we no longer use that directory.
```

**Step 2: Verify the change**

Run: `grep -A 8 "Step 8: Update .gitignore" /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/install-skill.md`
Expected: Mode-aware .gitignore handling.

**Step 3: Commit**

```bash
cd /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill
git add commands/install-skill.md
git commit -m "feat(install): simplify gitignore for new modes"
```

---

## Task 7: Update install-skill.md - Update Success Output

**Files:**
- Modify: `/home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/install-skill.md`

**Step 1: Replace Step 9 (success output) with mode-aware messages**

Find "### Step 9: Report success" and replace with:

```markdown
### Step 9: Report success

**Copy mode output:**
```
Installing skill from github.com/<owner>/<repo>...
├── Mode: Copy
├── Downloading: <skillPath>
├── Copying to: .claude/skills/<skill-name>
└── Status: Installed successfully

Skill "<skill-name>" is now available in this project.
```

**Submodule mode output:**
```
Installing skill from github.com/<owner>/<repo>...
├── Mode: Submodule
├── Adding submodule: .claude/submodules/<repo>
├── Creating symlink: .claude/skills/<skill-name>
└── Status: Installed successfully

Skill "<skill-name>" is now available in this project.

Note: Team members should run `git submodule update --init` after cloning.
```
```

**Step 2: Verify the change**

Run: `grep -A 20 "Step 9: Report success" /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/install-skill.md`
Expected: Both copy and submodule success messages.

**Step 3: Commit**

```bash
cd /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill
git add commands/install-skill.md
git commit -m "feat(install): mode-specific success messages"
```

---

## Task 8: Rewrite update-skill.md for New Modes

**Files:**
- Modify: `/home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/update-skill.md`

**Step 1: Replace entire file with new content**

```markdown
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
```

**Step 2: Verify the change**

Run: `head -50 /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/update-skill.md`
Expected: New frontmatter and structure.

**Step 3: Commit**

```bash
cd /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill
git add commands/update-skill.md
git commit -m "feat(update): support copy and submodule update modes"
```

---

## Task 9: Rewrite remove-skill.md for New Modes

**Files:**
- Modify: `/home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/remove-skill.md`

**Step 1: Replace entire file with new content**

```markdown
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
```

**Step 2: Verify the change**

Run: `head -50 /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/commands/remove-skill.md`
Expected: New frontmatter and mode-aware structure.

**Step 3: Commit**

```bash
cd /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill
git add commands/remove-skill.md
git commit -m "feat(remove): support copy and submodule removal"
```

---

## Task 10: Update README.md

**Files:**
- Modify: `/home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/README.md`

**Step 1: Read current README**

Run: `cat /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill/README.md`

**Step 2: Update README with new features**

Update to document:
- Two installation modes (copy and submodule)
- New command syntax with `--copy` and `--submodule` flags
- YAML registry format
- Team workflow for submodule mode

**Step 3: Commit**

```bash
cd /home/lagz0ne/.claude/plugins/marketplaces/local-install-skill
git add README.md
git commit -m "docs: update README for installation modes"
```

---

## Task 11: Final Integration Test

**Files:**
- Test all three commands in a test project

**Step 1: Create test project**

```bash
mkdir -p /tmp/test-skill-install
cd /tmp/test-skill-install
git init
```

**Step 2: Test copy mode install**

Run `/install-skill obra/superpowers:brainstorming --copy`

Verify:
- `.claude/skills/brainstorming/SKILL.md` exists (not a symlink)
- `.claude/local-plugins.yaml` has skill with `mode: copy`
- No `.claude/submodules/` directory

**Step 3: Test submodule mode install**

Run `/install-skill obra/superpowers:tdd --submodule`

Verify:
- `.claude/submodules/superpowers/` exists
- `.claude/skills/tdd` is a symlink to `../submodules/superpowers/skills/tdd`
- `.claude/local-plugins.yaml` has skill with `mode: submodule`
- `.gitmodules` has submodule entry

**Step 4: Test update**

Run `/update-skill --all`

Verify both modes update correctly.

**Step 5: Test remove**

Run `/remove-skill brainstorming` (copy mode)
Run `/remove-skill tdd` (submodule mode - should remove submodule too)

Verify cleanup is complete.

**Step 6: Clean up**

```bash
rm -rf /tmp/test-skill-install
```

---

## Summary

| Task | Description | Key Files |
|------|-------------|-----------|
| 1 | Add mode argument to frontmatter | install-skill.md |
| 2 | Add mode parsing section | install-skill.md |
| 3 | Add copy/submodule installation flows | install-skill.md |
| 4 | Mode-aware symlink creation | install-skill.md |
| 5 | YAML v2 registry format | install-skill.md |
| 6 | Simplify .gitignore handling | install-skill.md |
| 7 | Mode-specific success messages | install-skill.md |
| 8 | Rewrite update-skill for modes | update-skill.md |
| 9 | Rewrite remove-skill for modes | remove-skill.md |
| 10 | Update documentation | README.md |
| 11 | Integration testing | (test project) |

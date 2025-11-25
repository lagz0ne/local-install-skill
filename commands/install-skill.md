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

# Install Skill from GitHub

You are installing a Claude Code skill from a GitHub repository to project scope.

## Parse the Input

Parse the source argument: `$ARGUMENTS.source`

Supported formats:
- `user/repo` - Clone repo, look for skill at root or in `skills/` directory
- `user/repo:skill-name` - Install specific skill from `skills/<skill-name>/`
- `user/repo#branch` - Use specific branch
- `user/repo#branch:skill-name` - Branch + specific skill
- `user/repo --all` - Install all skills found in repo

Extract:
- `owner`: GitHub username/org
- `repo`: Repository name
- `branch`: Branch name (default: repo's default branch)
- `skillPath`: Path to skill within repo (optional)
- `installAll`: Boolean flag for --all

## Determine Installation Mode

Check for mode flags in `$ARGUMENTS.mode` or within `$ARGUMENTS.source`:

**Flag detection:**

1. First check `$ARGUMENTS.mode`:
   - If `--copy` or `copy`: `mode = "copy"`
   - If `--submodule` or `submodule`: `mode = "submodule"`

2. If not found, check if flag is appended to source string:
   - Split `$ARGUMENTS.source` by spaces
   - If last token is `--copy`: `mode = "copy"`, remove flag from source
   - If last token is `--submodule`: `mode = "submodule"`, remove flag from source

3. If no flag found, prompt user:

**Interactive prompt (when no flag):**

```
Use AskUserQuestion:
  question: "How would you like to install this skill?"
  header: "Install Mode"
  multiSelect: false
  options:
    - label: "Submodule (recommended)"
      description: "Full repo as git submodule, easy updates via git submodule update"
    - label: "Copy"
      description: "Only skill files copied, minimal footprint, re-run install to update"
```

Map user response to mode variable:
- "Submodule (recommended)" → `mode = "submodule"`
- "Copy" → `mode = "copy"`

**Default:** If user doesn't respond or cancels, default to `mode = "submodule"`.

The `mode` variable (value: `"copy"` or `"submodule"`) is used in subsequent installation steps.

## Installation Process

### Step 1: Setup directories

```bash
mkdir -p .claude/plugins/local
mkdir -p .claude/skills
```

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

### Step 4: Find skills to install

If `--all` flag:
- Look for `skills/` directory in cloned repo
- Find all subdirectories containing `SKILL.md`
- Collect list of skill names

If specific skill path:
- Check if `skills/<skill-name>/SKILL.md` exists
- If not, check if `<skill-name>/SKILL.md` exists at root
- If not found, report error with helpful message

If no skill specified:
- Check if repo root has `SKILL.md` (repo IS the skill)
- If not, check if `skills/` directory exists and has exactly one skill
- If multiple skills found, list them and ask user to specify

### Step 5: Validate each skill

For each skill to install:
1. Verify `SKILL.md` exists
2. Read frontmatter and validate:
   - `name` field exists (max 64 chars)
   - `description` field exists (max 1024 chars)
3. Warn if validation fails but continue

### Step 6: Create symlinks

For each skill:

Check if `.claude/skills/<skill-name>` already exists:
- If symlink pointing to same target: skip, already installed
- If symlink pointing elsewhere or regular directory: ask user to overwrite or skip

Create relative symlink:
```bash
ln -s ../plugins/local/<repo>/skills/<skill-name> .claude/skills/<skill-name>
```

Or if skill is at repo root:
```bash
ln -s ../plugins/local/<repo> .claude/skills/<skill-name>
```

### Step 7: Update registry

Read or create `.claude/local-plugins.json`:

```json
{
  "version": 1,
  "repos": {}
}
```

Add/update repo entry:
```json
{
  "version": 1,
  "repos": {
    "<repo>": {
      "source": "github.com/<owner>/<repo>",
      "branch": "<branch>",
      "clonedAt": "<ISO timestamp>",
      "gitCommitSha": "<current HEAD sha>",
      "installedSkills": ["<skill-name>", ...]
    }
  }
}
```

Get commit SHA:
```bash
git -C .claude/plugins/local/<repo> rev-parse HEAD
```

### Step 8: Update .gitignore

Ensure `.claude/plugins/local/` is in `.gitignore`:
```bash
echo ".claude/plugins/local/" >> .gitignore
```

Only add if not already present.

### Step 9: Report success

Output:
```
Installing skill from github.com/<owner>/<repo>...
├── Cloning: <owner>/<repo> (branch: <branch>)
├── Found skill: skills/<skill-name>/SKILL.md
├── Creating symlink: .claude/skills/<skill-name>
└── Status: Installed successfully

Skill "<skill-name>" is now available in this project.
```

## Error Handling

**Repository not found:**
```
Error: Repository not found: github.com/<owner>/<repo>

Check the repository URL and ensure it's public or you have access.
```

**No SKILL.md found:**
```
Error: No SKILL.md found at <owner>/<repo>:<path>

Expected structure:
  skill-name/
  └── SKILL.md (required)

Available skills in this repo:
  - skill-a (skills/skill-a/)
  - skill-b (skills/skill-b/)

Try: /install-skill <owner>/<repo>:skill-a
```

**Network error:**
```
Error: Failed to clone repository

Check your network connection and try again.
```

## Notes

- Always use relative symlinks so they work when repo is moved
- The `.claude/plugins/local/` directory should NOT be committed (add to .gitignore)
- The `.claude/skills/<name>` symlinks SHOULD be committed (team sharing)
- Team members need to run `/install-skill` after cloning to populate local repos

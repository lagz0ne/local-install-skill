# Skill Installation Modes - Design Document

## Overview

Enhance the local-install-skill plugin to support two installation modes: **Copy** and **Submodule**. This addresses git tree pollution from cloned repositories while giving users flexibility in how skills are managed.

## Problem Statement

The current implementation clones entire repositories to `.claude/plugins/local/`, which:
- Adds noise to git status and IDE file trees
- Includes files unrelated to the specific skill being installed
- Makes the `.claude` directory verbose and cluttered

## Solution

Two installation modes with per-skill choice:

| Mode | Storage | Git Impact | Best For |
|------|---------|------------|----------|
| **Copy** | `.claude/skills/<name>/` | Minimal - files committed directly | One-off skills, minimal footprint |
| **Submodule** | `.claude/submodules/<repo>/` + symlink | Submodule ref + symlink | Frequently updated skills, team sync |

**Default:** Submodule (easier updates, cleaner git management)

## Directory Structure

### Copy Mode

```
project/
└── .claude/
    ├── skills/
    │   └── brainstorming/          # Copied skill files (committed)
    │       └── SKILL.md
    └── local-plugins.yaml          # Registry
```

### Submodule Mode

```
project/
└── .claude/
    ├── submodules/
    │   └── superpowers/            # Git submodule
    │       └── skills/
    │           └── brainstorming/
    ├── skills/
    │   └── brainstorming -> ../submodules/superpowers/skills/brainstorming
    └── local-plugins.yaml          # Registry
```

## Command Interface

### /install-skill

**Syntax:**
```bash
/install-skill user/repo:skill-name           # Interactive prompt for mode
/install-skill user/repo:skill-name --submodule  # Skip prompt, use submodule
/install-skill user/repo:skill-name --copy       # Skip prompt, use copy
/install-skill user/repo --all                   # Install all skills (prompts for mode)
/install-skill user/repo --all --copy            # Install all via copy
```

**Interactive prompt (when no flag provided):**
```
Installation mode?
  [1] Submodule (default) - Full repo as git submodule, easy updates
  [2] Copy - Only skill files, minimal footprint
>
```

### /update-skill

**Syntax:**
```bash
/update-skill repo-name              # Update submodule repo
/update-skill skill-name --copy      # Re-download and overwrite (copy mode)
/update-skill --all                  # Update all submodule repos
```

### /remove-skill

**Syntax:**
```bash
/remove-skill skill-name             # Remove skill, clean up if last from repo
```

## Registry Format

**File:** `.claude/local-plugins.yaml`

```yaml
version: 2

skills:
  brainstorming:
    mode: submodule
    source: github.com/user/superpowers
    repo: superpowers
    branch: main
    skillPath: skills/brainstorming
    installedAt: 2025-11-25T10:00:00Z
    commitSha: abc123...

  my-custom-skill:
    mode: copy
    source: github.com/user/other-repo
    branch: main
    skillPath: skills/my-custom-skill
    installedAt: 2025-11-25T11:00:00Z
    commitSha: def456...

submodules:
  superpowers:
    source: github.com/user/superpowers
    path: .claude/submodules/superpowers
    skills:
      - brainstorming
      - tdd
```

## Installation Flows

### Copy Mode Flow

1. Create temp directory
2. `git clone --depth=1 <url>` to temp
3. Copy `<temp>/<skillPath>/` to `.claude/skills/<name>/`
4. Delete temp directory
5. Record in `local-plugins.yaml` with `mode: copy`

### Submodule Mode Flow

1. Check if `.claude/submodules/<repo>/` exists
2. If not: `git submodule add <url> .claude/submodules/<repo>`
3. If exists: verify it's the same source
4. Create symlink: `.claude/skills/<name>` → `../submodules/<repo>/<skillPath>`
5. Record in `local-plugins.yaml` with `mode: submodule`
6. Update `submodules` section with skill reference

### Update Flow

**Copy mode:**
1. Clone to temp (same as install)
2. Remove existing `.claude/skills/<name>/`
3. Copy fresh files
4. Update `commitSha` in registry

**Submodule mode:**
1. `git -C .claude/submodules/<repo> fetch`
2. `git -C .claude/submodules/<repo> pull origin <branch>`
3. Update `commitSha` in registry

### Remove Flow

1. Look up skill in registry
2. If copy mode:
   - `rm -rf .claude/skills/<name>/`
3. If submodule mode:
   - `rm .claude/skills/<name>` (symlink)
   - Remove skill from `submodules.<repo>.skills` list
   - If no more skills use this submodule:
     - `git submodule deinit .claude/submodules/<repo>`
     - `git rm .claude/submodules/<repo>`
     - Remove entry from `submodules` section
4. Remove skill entry from `skills` section
5. Write updated registry

## Migration from v1

When encountering `local-plugins.json` (v1):
1. Read existing entries
2. Convert to v2 YAML format
3. Existing installs assumed to be legacy "clone" mode
4. Offer to migrate to submodule or copy mode
5. Write `local-plugins.yaml`, remove `.json`

## Error Handling

| Scenario | Behavior |
|----------|----------|
| Not a git repo (submodule mode) | Error: "Submodule mode requires a git repository" |
| Submodule already exists, different source | Error with option to remove existing |
| Skill already installed | Ask to overwrite or skip |
| Network failure | Clean error, suggest retry |
| Invalid SKILL.md | Warn but continue |

## Output Format

**Install success (submodule):**
```
Installing skill from github.com/user/superpowers...
├── Mode: Submodule
├── Adding submodule: .claude/submodules/superpowers
├── Creating symlink: .claude/skills/brainstorming
└── Status: Installed successfully

Skill "brainstorming" is now available.
```

**Install success (copy):**
```
Installing skill from github.com/user/superpowers...
├── Mode: Copy
├── Downloading: skills/brainstorming
├── Copying to: .claude/skills/brainstorming
└── Status: Installed successfully

Skill "brainstorming" is now available.
```

## Git Integration

- **Copy mode:** `.claude/skills/<name>/` committed directly
- **Submodule mode:**
  - `.gitmodules` updated automatically by git
  - `.claude/skills/<name>` symlink committed
  - Team members run `git submodule update --init` after clone

## Files Changed

- `commands/install-skill.md` - Add mode selection logic
- `commands/update-skill.md` - Handle both modes
- `commands/remove-skill.md` - Handle submodule cleanup
- Registry format: `.json` → `.yaml`, schema v1 → v2

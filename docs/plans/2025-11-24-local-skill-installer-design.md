# Local Skill Installer - Design Document

## Overview

A Claude Code plugin that enables project-scoped skill installation from GitHub repositories. Unlike the marketplace which installs to user scope (`~/.claude/plugins/`), this tool installs skills directly into the project's `.claude/skills/` directory for version-controlled, team-shared skills.

## Problem Statement

Claude Code's marketplace only supports user-scope plugin installation. Teams need:
1. Skills version-controlled with the project
2. No need to publish to a marketplace
3. Automatic skill availability when team members clone the repo

## Solution

Clone repositories to `.claude/plugins/local/` and symlink specific skills to `.claude/skills/`.

## Architecture

### Directory Structure

```
project/
├── .claude/
│   ├── plugins/
│   │   └── local/                          # Full clones live here
│   │       └── <repo-name>/                # git clone of repo
│   │           ├── .git/
│   │           ├── skills/
│   │           │   ├── skill-a/
│   │           │   └── skill-b/
│   │           └── ...
│   ├── skills/
│   │   ├── skill-a -> ../plugins/local/<repo>/skills/skill-a
│   │   └── skill-b -> ../plugins/local/<repo>/skills/skill-b
│   └── local-plugins.json                  # Registry tracking installs
```

### Registry Format (`local-plugins.json`)

```json
{
  "version": 1,
  "repos": {
    "<repo-name>": {
      "source": "github.com/owner/repo",
      "branch": "main",
      "clonedAt": "2025-11-24T10:00:00Z",
      "gitCommitSha": "abc123...",
      "installedSkills": ["skill-a", "skill-b"]
    }
  }
}
```

## Commands

### `/install-skill`

Install a skill from a GitHub repository.

**Syntax:**
```bash
/install-skill user/repo:skill-name         # Install specific skill
/install-skill user/repo --all              # Install all skills from repo
/install-skill user/repo#branch:skill-name  # Specific branch
```

**Process:**
1. Parse input (owner, repo, branch, skill path)
2. Check if repo already cloned in `.claude/plugins/local/`
3. If not, `git clone --depth=1` to `.claude/plugins/local/<repo>/`
4. Validate skill exists and has SKILL.md
5. Create symlink in `.claude/skills/`
6. Update `local-plugins.json` registry
7. Report success

### `/update-skill`

Update a previously installed skill/repo.

**Syntax:**
```bash
/update-skill repo-name                     # Update specific repo
/update-skill --all                         # Update all repos
```

**Process:**
1. `git pull` in the repo directory
2. Verify symlinks still valid
3. Update gitCommitSha in registry
4. Report changes

### `/remove-skill`

Remove an installed skill.

**Syntax:**
```bash
/remove-skill skill-name                    # Remove skill symlink
/remove-skill skill-name --purge            # Also remove repo if no other skills installed
```

**Process:**
1. Remove symlink from `.claude/skills/`
2. Update registry
3. If `--purge` and no other skills from repo, remove repo directory

## Error Handling

| Scenario | Behavior |
|----------|----------|
| Skill already exists | Ask to overwrite or skip |
| No SKILL.md found | Error with helpful message |
| Invalid repo/path | Error with format examples |
| No `.claude/` dir | Create it automatically |
| Network failure | Clean error, suggest retry |
| Symlink target missing | Warn and offer to reinstall |

## Validation

Post-install validation:
1. Check SKILL.md exists
2. Validate frontmatter has required `name` and `description`
3. Warn if description > 1024 chars or name > 64 chars

## Output Format

**Success:**
```
Installing skill from github.com/user/repo...
├── Cloning: user/repo (branch: main)
├── Found skill: skills/brainstorming/SKILL.md
├── Creating symlink: .claude/skills/brainstorming
└── Status: Installed successfully

Skill "brainstorming" is now available in this project.
```

**Error:**
```
Error: No SKILL.md found at user/repo:path/to/skill

Expected structure:
  skill-name/
  └── SKILL.md (required)

Check the repository path and try again.
```

## Git Integration

- Symlinks should be committed to git (team sharing)
- `.claude/plugins/local/` should be in `.gitignore`
- Team members run `/install-skill` after clone to populate local repos

## Future Considerations

- Support for GitLab/Bitbucket sources
- Lock file for reproducible installs
- Automatic installation hook on project open

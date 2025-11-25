# local-install-skill

A Claude Code plugin that installs skills from GitHub repositories to project scope.

## Why?

Claude Code's marketplace installs plugins to user scope (`~/.claude/plugins/`). This plugin enables **project-scoped** skill installation:

- Skills version-controlled with your project
- No need to publish to a marketplace
- Team members get skills automatically when they clone the repo

## Installation

```bash
# Add this plugin to your Claude Code installation
/plugin marketplace add lagz0ne/local-install-skill
/plugin install local-install-skill@lagz0ne
```

## Installation Modes

Choose how skills are stored in your project:

| Mode | Storage | Best For |
|------|---------|----------|
| **Submodule** (default) | Git submodule + symlink | Teams, frequently updated skills |
| **Copy** | Direct file copy | Minimal footprint, one-off skills |

## Usage

### Install a skill

```bash
# Install with interactive mode selection (default: submodule)
/install-skill user/repo:skill-name

# Explicitly use submodule mode
/install-skill user/repo:skill-name --submodule

# Use copy mode (files copied directly)
/install-skill user/repo:skill-name --copy

# Install from specific branch
/install-skill user/repo#develop:skill-name

# Install all skills from a repo
/install-skill user/repo --all
```

### Update installed skills

```bash
# Update a submodule repo (updates all skills from it)
/update-skill repo-name

# Update a copy-mode skill (re-downloads)
/update-skill skill-name

# Update all
/update-skill --all
```

### Remove a skill

```bash
# Remove skill (cleans up submodule if no other skills use it)
/remove-skill skill-name
```

## How it works

### Submodule Mode (default)

```
.claude/
├── submodules/              # Git submodules
│   └── superpowers/         # Full repo as submodule
│       └── skills/
│           └── brainstorming/
├── skills/                  # Symlinks (committed)
│   └── brainstorming -> ../submodules/superpowers/skills/brainstorming
└── local-plugins.yaml       # Registry (committed)
```

### Copy Mode

```
.claude/
├── skills/                  # Copied files (committed)
│   └── my-skill/
│       └── SKILL.md
└── local-plugins.yaml       # Registry (committed)
```

## Registry Format

Skills are tracked in `.claude/local-plugins.yaml`:

```yaml
version: 2

skills:
  brainstorming:
    mode: submodule
    source: github.com/user/superpowers
    repo: superpowers
    branch: main
    skillPath: skills/brainstorming
    commitSha: abc123...

  my-skill:
    mode: copy
    source: github.com/user/other-repo
    branch: main
    skillPath: skills/my-skill
    commitSha: def456...

submodules:
  superpowers:
    source: github.com/user/superpowers
    path: .claude/submodules/superpowers
    skills:
      - brainstorming
```

## Team Workflow

### For submodule mode:

1. One team member installs: `/install-skill org/repo:skill-name`
2. Commit: `git add .claude/skills .claude/local-plugins.yaml .gitmodules`
3. Push to remote
4. Other team members: `git pull && git submodule update --init`

### For copy mode:

1. One team member installs: `/install-skill org/repo:skill-name --copy`
2. Commit: `git add .claude/skills .claude/local-plugins.yaml`
3. Push to remote
4. Other team members: `git pull` (skills are already in the files)

## License

MIT

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

Or install directly to a project:
```bash
git clone https://github.com/lagz0ne/local-install-skill.git .claude/plugins/local/local-install-skill
```

## Usage

### Install a skill

```bash
# Install specific skill from a repo
/install-skill user/repo:skill-name

# Install from specific branch
/install-skill user/repo#develop:skill-name

# Install all skills from a repo
/install-skill user/repo --all
```

### Update installed skills

```bash
# Update specific repo
/update-skill repo-name

# Update all repos
/update-skill --all
```

### Remove a skill

```bash
# Remove skill (keep repo)
/remove-skill skill-name

# Remove skill and repo (if no other skills installed)
/remove-skill skill-name --purge
```

## How it works

1. **Clone**: Full git clone to `.claude/plugins/local/<repo>/`
2. **Symlink**: Creates symlink in `.claude/skills/<skill-name>`
3. **Registry**: Tracks installs in `.claude/local-plugins.json`

```
.claude/
├── plugins/local/           # Git clones (gitignored)
│   └── some-repo/
│       └── skills/
│           └── cool-skill/
├── skills/                  # Symlinks (committed)
│   └── cool-skill -> ../plugins/local/some-repo/skills/cool-skill
└── local-plugins.json       # Registry (committed)
```

## Team workflow

1. One team member installs skills: `/install-skill org/repo:skill-name`
2. Commit the symlinks and registry: `git add .claude/skills .claude/local-plugins.json`
3. Other team members clone and run: `/install-skill` (reads registry, clones repos)

## License

MIT

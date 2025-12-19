# claude-sandbox

> Test workspace for the [.claude](.claude/) development environment.

## Purpose

Development and testing workspace for the `.claude` Docker-ized Claude Code environment. This repository serves as the parent project that includes `.claude` as a git submodule.

## Quick Start

```bash
# Clone with submodules
git clone --recursive https://github.com/brianclaridge/claude-sandbox.git
cd claude-sandbox

# Or if already cloned
git submodule update --init --recursive

# Start the environment
task claude
```

## Structure

```
/workspace/
├── .claude/           # Development environment (submodule)
│   ├── agents/        # Specialized agents
│   ├── skills/        # Automation skills
│   ├── rules/         # Behavioral rules
│   ├── docs/          # Documentation
│   └── ...
├── plans/             # Implementation plans (created here)
├── CLAUDE.md          # Claude Code workflow guide
├── Taskfile.yml       # Task definitions
└── README.md
```

## Documentation

| Resource | Purpose |
|----------|---------|
| [CLAUDE.md](CLAUDE.md) | Claude Code workflow for this workspace |
| [.claude/README.md](.claude/README.md) | Environment setup and overview |
| [.claude/CLAUDE.md](.claude/CLAUDE.md) | Submodule-specific guidance |
| [.claude/docs/](.claude/docs/) | Full documentation |

## Key Concepts

### Plans Location

Implementation plans for this workspace are created in:
```
/workspace/plans/
```

Plans for `.claude` submodule development go in `.claude/plans/`.

### Available Commands

| Command | Action |
|---------|--------|
| `task claude` | Start environment |
| `task claude:fetch` | Update submodule |
| `/analyze` | Analyze codebase |
| `/gitops` | Commit workflow |
| `/health` | Validate setup |

## Submodule Management

### Update

```bash
task claude:fetch
# or
git submodule update --remote --merge .claude
```

### Reset

```bash
git -C .claude fetch origin
git -C .claude reset --hard origin/main
git -C .claude clean -fd
```

### Remove

```bash
git submodule deinit -f .claude
rm -rf .git/modules/.claude
git rm -f .claude
```

## License

MIT

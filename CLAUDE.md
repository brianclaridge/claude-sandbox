# Claude Code Guide - claude-sandbox

> Workflow guide for Claude Code in this workspace.

## About This Project

Test workspace for the [.claude](.claude/) development environment.

## Submodule: .claude

The `.claude/` directory is a git submodule providing:
- Specialized agents for complex tasks
- Skills for automated workflows
- Behavioral rules governing Claude's actions
- Docker-ized development environment

See [.claude/CLAUDE.md](.claude/CLAUDE.md) for submodule-specific guidance.

## Rules Applied

Rules from `.claude/rules/` are automatically loaded. Key rules:

| Rule | Effect |
|------|--------|
| 010 | Session starts with project analysis |
| 040 | Plans created in `/workspace/plans/` (not `.claude/plans/`) |
| 030 | Agents considered for every request |

## Quick Reference

| Command | Action |
|---------|--------|
| `/hello` | Test greeting |
| `/analyze` | Analyze codebase |
| `/cloud-auth` | AWS/GCP authentication |
| `/playwright` | Browser automation |
| `/gitops` | Commit changes |
| `/health` | Validate environment |

## Plan Location

**Important**: Implementation plans for this workspace go in:
```
/workspace/plans/
```

NOT in `.claude/plans/` (which is for submodule development).

## Git Workflow

After completing work:
1. All TODOs marked complete
2. git-manager skill invoked automatically
3. Commit to current branch
4. Enter plan mode for next task

## Documentation

| Resource | Location |
|----------|----------|
| Environment Setup | [.claude/README.md](.claude/README.md) |
| Claude Workflow | [.claude/CLAUDE.md](.claude/CLAUDE.md) |
| Full Documentation | [.claude/docs/](.claude/docs/) |
| Agents | [.claude/docs/agents/](.claude/docs/agents/) |
| Skills | [.claude/docs/skills/](.claude/docs/skills/) |
| Rules | [.claude/docs/rules/](.claude/docs/rules/) |

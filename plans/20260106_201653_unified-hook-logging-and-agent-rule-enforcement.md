# Plan: Unified Hook Logging and Agent Rule Enforcement

**Affects:** `/workspace/.claude/apps/src/claude_apps/shared/hook_logging/`, `/workspace/.claude/apps/src/claude_apps/hooks/*/`, `/workspace/.claude/agents/*.md`

---

## Summary

Two related improvements:
1. **Hook Logging**: Add unified structlog + loguru logging to all 8 hooks for human-readable output
2. **Agent Rules**: Add rule references to 11 agents (28 missing rule-agent connections)

---

## Phase 1: Unified Hook Logging

### Architecture: structlog + loguru Bridge

```
Hook Code
    │
    ▼
structlog (structured context binding)
    │
    ▼
LoguruBridge processor
    │
    ▼
loguru (beautiful stderr output)
```

**Rationale:**
- structlog: Structured context, event-based logging
- loguru: Colors, formatting, human-readable output
- All output to stderr (stdout reserved for hook JSON protocol)

### New Module: `/workspace/.claude/apps/src/claude_apps/shared/hook_logging/`

```
hook_logging/
├── __init__.py      # get_hook_logger() entry point
├── bridge.py        # LoguruBridge processor
└── config.py        # Level/format configuration
```

### Core API

```python
from claude_apps.shared.hook_logging import get_hook_logger

logger = get_hook_logger("plan_distributor")
logger.info("plan_distributed", plan_file="/path/to/plan.md", affects=["/src/"])
```

### Output Format (stderr)

```
18:30:45 | INFO     | plan_distributor | plan_distributed plan_file=/path/to/plan.md
```

### Hook Migration Order

| # | Hook | Complexity | Current State |
|---|------|------------|---------------|
| 1 | `plan_distributor` | Simple | Minimal (traceback only) |
| 2 | `implementation_start_detector` | Simple | Minimal |
| 3 | `session_context_injector` | Medium | Inline structlog |
| 4 | `submodule_auto_updater` | Medium | Inline structlog |
| 5 | `cloud_auth_prompt` | Medium | Inline structlog |
| 6 | `changelog_monitor` | Medium | Inline structlog |
| 7 | `rules_loader` | Complex | Dedicated logger.py |
| 8 | `playwright_healer` | Complex | Dedicated logger.py |

---

## Phase 2: Agent Rule Enforcement

### Gap Matrix (28 missing connections)

| Agent | 000 | 020 | 030 | 040 | 060 | 070 | 080 |
|-------|-----|-----|-----|-----|-----|-----|-----|
| project-analysis | ADD | ADD | - | Has | - | - | - |
| agent-builder | ADD | ADD | ADD | ADD | ADD | ADD | ADD |
| skill-builder | ADD | ADD | ADD | ADD | ADD | ADD | ADD |
| stack-manager | ADD | ADD | ADD | - | ADD | ADD | ADD |
| gitops | ADD | ADD | ADD | - | - | - | - |
| browser-automation | ADD | ADD | - | - | ADD | - | - |
| rule-builder | ADD | ADD | - | ADD | - | ADD | ADD |
| taskfile-manager | ADD | ADD | - | - | - | - | ADD |
| gomplate-manager | ADD | ADD | - | - | - | - | ADD |
| health-check | ADD | ADD | - | - | - | - | - |

**Legend:**
- 000: rule-follower (core adherence)
- 020: persona (Spock)
- 030: agent analysis tables
- 040: planning workflow
- 060: context7 for libraries
- 070: backward-compat default
- 080: AskUserQuestion usage

### Rule Compliance Section Template

Add after frontmatter in each agent:

```markdown
## Rule Compliance

| Rule | Relevance |
|------|-----------|
| 000 (rule-follower) | Core directive adherence |
| 020 (persona) | Spock persona in responses |
| [rule] | [relevance] |
```

---

## Tasks

### Phase 1: Hook Logging Foundation
- [ ] Create `/workspace/.claude/apps/src/claude_apps/shared/hook_logging/__init__.py`
- [ ] Create `/workspace/.claude/apps/src/claude_apps/shared/hook_logging/bridge.py`
- [ ] Create `/workspace/.claude/apps/src/claude_apps/shared/hook_logging/config.py`
- [ ] Add unit tests for hook_logging module

### Phase 2: Hook Migration
- [ ] Migrate `plan_distributor/__main__.py`
- [ ] Migrate `implementation_start_detector/__main__.py`
- [ ] Migrate `session_context_injector/__main__.py`
- [ ] Migrate `submodule_auto_updater/__main__.py`
- [ ] Migrate `cloud_auth_prompt/__main__.py`
- [ ] Migrate `changelog_monitor/__main__.py`
- [ ] Migrate `rules_loader/__main__.py` (remove logger.py)
- [ ] Migrate `playwright_healer/__main__.py` (remove logger.py)

### Phase 3: Agent Updates
- [ ] Update `_template.md` with Rule Compliance section
- [ ] Update `project-analysis.md` (add 000, 020)
- [ ] Update `agent-builder.md` (add 000, 020, 030, 040, 060, 070, 080)
- [ ] Update `skill-builder.md` (add 000, 020, 030, 040, 060, 070, 080)
- [ ] Update `stack-manager.md` (add 000, 020, 030, 060, 070, 080)
- [ ] Update `gitops.md` (add 000, 020, 030)
- [ ] Update `browser-automation.md` (add 000, 020, 060)
- [ ] Update `rule-builder.md` (add 000, 020, 040, 070, 080)
- [ ] Update `taskfile-manager.md` (add 000, 020, 080)
- [ ] Update `gomplate-manager.md` (add 000, 020, 080)
- [ ] Update `health-check.md` (add 000, 020)

### Phase 4: Validation
- [ ] Run all hook tests
- [ ] Verify stderr output (not stdout)
- [ ] Verify hook JSON protocol preserved

---

## Critical Files

**New:**
- `/workspace/.claude/apps/src/claude_apps/shared/hook_logging/__init__.py`
- `/workspace/.claude/apps/src/claude_apps/shared/hook_logging/bridge.py`

**Modified (Hooks):**
- `/workspace/.claude/apps/src/claude_apps/hooks/*/__main__.py` (8 files)

**Modified (Agents):**
- `/workspace/.claude/agents/*.md` (10 files + template)

**Deprecated:**
- `/workspace/.claude/apps/src/claude_apps/hooks/rules_loader/logger.py`
- `/workspace/.claude/apps/src/claude_apps/hooks/playwright_healer/logger.py`

---

## Implementation Details

### hook_logging/__init__.py

```python
"""Unified logging for Claude Code hooks (structlog + loguru)."""

import sys
import structlog
from loguru import logger as loguru_logger
from .bridge import LoguruBridge

def get_hook_logger(hook_name: str, level: str = "INFO") -> structlog.BoundLogger:
    """Get configured logger for a hook.

    Args:
        hook_name: Hook identifier for context
        level: Log level (DEBUG, INFO, WARNING, ERROR)

    Returns:
        Configured structlog BoundLogger
    """
    # Configure loguru to stderr with pretty format
    loguru_logger.remove()
    loguru_logger.add(
        sys.stderr,
        format="<green>{time:HH:mm:ss}</green> | <level>{level: <8}</level> | <cyan>{extra[hook]}</cyan> | {message}",
        level=level,
        colorize=True,
    )

    # Configure structlog with loguru bridge
    structlog.configure(
        processors=[
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.format_exc_info,
            LoguruBridge(hook_name=hook_name),
        ],
        wrapper_class=structlog.stdlib.BoundLogger,
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=False,
    )

    return structlog.get_logger().bind(hook=hook_name)
```

### hook_logging/bridge.py

```python
"""Bridge processor routing structlog events to loguru."""

from typing import Any
from loguru import logger as loguru_logger

class LoguruBridge:
    """Structlog processor that routes to loguru."""

    def __init__(self, hook_name: str):
        self.hook_name = hook_name

    def __call__(self, logger: Any, method_name: str, event_dict: dict) -> str:
        event = event_dict.pop("event", "")
        level = event_dict.pop("level", method_name)
        event_dict.pop("timestamp", None)

        # Format context as key=value pairs
        context = " ".join(f"{k}={v}" for k, v in event_dict.items() if k != "hook")
        message = f"{event} {context}".strip()

        getattr(loguru_logger.bind(hook=self.hook_name), level)(message)
        return ""
```

---

## Success Criteria

1. All 8 hooks use unified `get_hook_logger()` API
2. Human-readable colored output on stderr
3. Hook JSON stdout protocol unchanged
4. All 10 agents have Rule Compliance sections
5. All existing tests pass
6. New hook_logging tests pass

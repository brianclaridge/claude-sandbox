# Plan: Implementation Start Detection Hook

**Affects:** `/workspace/.claude/apps/src/claude_apps/hooks/plan_distributor/`, `/workspace/.claude/apps/src/claude_apps/hooks/implementation_start_detector/`, `/workspace/.claude/config/templates/settings.json`, `/workspace/.claude/apps/tests/hooks/`

---

## Summary

Implement a sentinel file pattern to detect when implementation begins after plan approval. This provides a workaround for the missing "plan approved, implementation starting" hook event in Claude Code.

## Architecture

```
PostToolUse:ExitPlanMode
    │
    ├─► plan_distributor (existing, extended)
    │       ├─► distribute plan to ./plans/
    │       └─► CREATE sentinel: .data/pending_implementation.json
    │
    └─► (plan mode ends, implementation begins)

PreToolUse:Edit|Write (new hook)
    │
    └─► implementation_start_detector (new)
            ├─► CHECK sentinel exists
            ├─► IF EXISTS: First edit after plan approval
            │       ├─► Log/emit "implementation started" event
            │       └─► DELETE sentinel
            └─► IF NOT: Normal edit, pass through
```

## Tasks

### Phase 1: Extend plan_distributor

- [ ] Add sentinel file creation to `distributor.py`
- [ ] Define sentinel schema in new `sentinel.py` module
- [ ] Update `__main__.py` to invoke sentinel creation after distribution
- [ ] Add sentinel file path to shared constants

### Phase 2: Create implementation_start_detector hook

- [ ] Create `/workspace/.claude/apps/src/claude_apps/hooks/implementation_start_detector/` directory
- [ ] Implement `__init__.py` with module exports
- [ ] Implement `__main__.py` with hook entry point
- [ ] Implement `detector.py` with sentinel check/clear logic

### Phase 3: Register new hook

- [ ] Update `/workspace/.claude/config/templates/settings.json` PreToolUse section
- [ ] Add matcher for `Edit|Write` tools
- [ ] Register implementation_start_detector command

### Phase 4: Tests

- [ ] Add tests for sentinel creation in plan_distributor
- [ ] Add tests for implementation_start_detector hook
- [ ] Add integration test for full flow

---

## Implementation Details

### Sentinel File Schema

**Location:** `${CLAUDE_DATA_PATH}/pending_implementation.json`

```json
{
  "plan_file": "/workspace/plans/20260106_120000_feature-name.md",
  "distributed_at": "2026-01-06T12:00:00.000Z",
  "affects": ["/workspace/src/", "/workspace/tests/"],
  "cwd": "/workspace",
  "session_id": "abc123"
}
```

### New Module: sentinel.py

```python
"""Sentinel file management for plan implementation tracking."""

from dataclasses import dataclass
from datetime import datetime
from pathlib import Path
import json
import os

SENTINEL_FILENAME = "pending_implementation.json"

@dataclass
class ImplementationSentinel:
    plan_file: str
    distributed_at: str
    affects: list[str]
    cwd: str
    session_id: str | None = None

def get_sentinel_path() -> Path:
    """Get sentinel file path from environment."""
    data_path = os.environ.get("CLAUDE_DATA_PATH", ".claude/.data")
    return Path(data_path) / SENTINEL_FILENAME

def create_sentinel(
    plan_file: str,
    affects: list[str],
    cwd: str,
    session_id: str | None = None
) -> Path:
    """Create sentinel file indicating plan awaiting implementation."""
    sentinel = ImplementationSentinel(
        plan_file=plan_file,
        distributed_at=datetime.now().isoformat(),
        affects=affects,
        cwd=cwd,
        session_id=session_id
    )

    path = get_sentinel_path()
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(json.dumps(sentinel.__dict__, indent=2))
    return path

def read_sentinel(max_age_seconds: int = 3600) -> ImplementationSentinel | None:
    """Read sentinel file if exists and not stale.

    Args:
        max_age_seconds: Maximum age in seconds (default: 1 hour).
                        Sentinels older than this are auto-cleared.

    Returns:
        Sentinel if valid, None if missing or stale.
    """
    path = get_sentinel_path()
    if not path.exists():
        return None

    data = json.loads(path.read_text())
    sentinel = ImplementationSentinel(**data)

    # TTL check - auto-clear stale sentinels
    created = datetime.fromisoformat(sentinel.distributed_at)
    age = (datetime.now() - created).total_seconds()
    if age > max_age_seconds:
        clear_sentinel()
        return None

    return sentinel

def clear_sentinel() -> bool:
    """Remove sentinel file. Returns True if file existed."""
    path = get_sentinel_path()
    if path.exists():
        path.unlink()
        return True
    return False
```

### Hook: implementation_start_detector/__main__.py

```python
"""Entry point for implementation_start_detector hook.

Triggers on PreToolUse for Edit|Write tools. Detects first edit
after plan approval by checking for sentinel file.
"""

import json
import sys
from typing import Any

from ..plan_distributor.sentinel import read_sentinel, clear_sentinel

def process_hook_event(hook_data: dict[str, Any]) -> dict[str, Any]:
    """Process PreToolUse hook for Edit|Write tools."""
    response: dict[str, Any] = {
        "continue": True,
        "suppressOutput": False
    }

    tool_name = hook_data.get("tool_name", "")

    # Only process Edit and Write tools
    if tool_name not in ("Edit", "Write"):
        return response

    # Check for pending implementation sentinel
    sentinel = read_sentinel()
    if sentinel is None:
        return response

    # First edit after plan approval detected!
    clear_sentinel()

    response["hookSpecificOutput"] = {
        "hookEventName": "PreToolUse",
        "additionalContext": (
            f"[IMPLEMENTATION START] Beginning implementation of plan: "
            f"{sentinel.plan_file}"
        )
    }

    return response

def main() -> int:
    """Main entry point."""
    try:
        data = sys.stdin.read().strip()
        if not data:
            print(json.dumps({"continue": True}))
            return 0

        hook_data = json.loads(data)
        if isinstance(hook_data, list):
            hook_data = hook_data[0] if hook_data else {}

        response = process_hook_event(hook_data)
        print(json.dumps(response))
        return 0

    except Exception as e:
        print(json.dumps({
            "continue": True,
            "hookSpecificOutput": {
                "additionalContext": f"[IMPLEMENTATION START DETECTOR] Error: {e}"
            }
        }))
        return 0

if __name__ == "__main__":
    sys.exit(main())
```

### Settings.json PreToolUse Update

```json
"PreToolUse": [
  {
    "matcher": "Edit|Write",
    "hooks": [
      {
        "type": "command",
        "command": "uv run --directory {{ .Env.CLAUDE_PATH }} python -m claude_apps.hooks.implementation_start_detector"
      }
    ]
  },
  {
    "matcher": "*",
    "hooks": [
      {
        "type": "command",
        "command": "jq '.' >&2; echo '{\"hook_event_name\":\"PreToolUse\"}'"
      },
      {
        "type": "command",
        "command": "uv run --directory {{ .Env.CLAUDE_PATH }} python -m claude_apps.hooks.logger"
      }
    ]
  }
]
```

### plan_distributor/distributor.py Changes

Add after successful distribution:

```python
from .sentinel import create_sentinel

# In distribute_plan(), after result.destinations populated:
if result.destinations:
    # Create sentinel for implementation tracking
    create_sentinel(
        plan_file=str(result.destinations[0]),
        affects=result.affects or [],
        cwd=hook_cwd or os.getcwd(),
        session_id=None  # Could extract from hook_data if available
    )
```

---

## File Structure After Implementation

```
.claude/apps/src/claude_apps/hooks/
├── plan_distributor/
│   ├── __init__.py
│   ├── __main__.py
│   ├── distributor.py      # MODIFIED - add sentinel creation
│   ├── parser.py
│   └── sentinel.py         # NEW - sentinel management
├── implementation_start_detector/
│   ├── __init__.py         # NEW
│   └── __main__.py         # NEW - hook entry point
└── ...
```

---

## Testing Strategy

### Unit Tests

1. `test_sentinel.py` - sentinel CRUD operations
2. `test_implementation_start_detector.py` - hook behavior

### Integration Test

```python
def test_full_flow():
    """Test plan distribution creates sentinel, first edit clears it."""
    # 1. Simulate ExitPlanMode PostToolUse
    # 2. Verify sentinel created
    # 3. Simulate Edit PreToolUse
    # 4. Verify sentinel cleared
    # 5. Simulate another Edit
    # 6. Verify no sentinel interaction
```

---

## Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Sentinel file not cleaned up on error | **IMPLEMENTED**: TTL check auto-clears if >1 hour old |
| Multiple plans approved rapidly | Sentinel overwrites - only tracks latest |
| Edit without prior plan | No sentinel = normal pass-through |

---

## Success Criteria

1. `PostToolUse:ExitPlanMode` creates sentinel file
2. First `PreToolUse:Edit|Write` detects and clears sentinel
3. Hook output indicates "implementation started"
4. Subsequent edits have no sentinel interaction
5. All existing tests pass
6. New tests for sentinel and detector pass

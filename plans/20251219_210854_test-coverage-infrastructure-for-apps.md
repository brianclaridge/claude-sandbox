# Plan: Test Coverage Infrastructure for apps/

**Affects:** `/workspace/.claude/pyproject.toml`, `/workspace/.claude/apps/tests/`, `/workspace/.claude/.gitignore`, `/workspace/.claude/Taskfile.yml`

---

## Summary

Implement phased test coverage to reach 100% for 147 Python files across `shared/`, `skills/`, and `hooks/` modules.

## User Requirements

| Requirement | Choice |
|-------------|--------|
| Framework | pytest |
| Reports | Terminal only |
| Modes | All (fail-fast, parallel, verbose, coverage-only) |
| Priority | Critical path: shared/ → skills/ → hooks/ |
| Mocking | pytest-mock + responses |
| Threshold | pyproject.toml enforcement |

## Current State

- **pyproject.toml**: Has pytest + pytest-cov + moto configured
- **apps/tests/**: Empty `__init__.py` placeholders only
- **.gitignore**: Has `.pytest_cache`, missing coverage output

---

## Phase 0: Infrastructure Setup

### 0.1 Update pyproject.toml

Add to `[project.optional-dependencies]`:
```toml
dev = [
    "moto>=5.0.0",
    "pytest>=8.0.0",
    "pytest-cov>=4.0.0",
    "pytest-mock>=3.12.0",      # ADD
    "pytest-xdist>=3.5.0",      # ADD - parallel execution
    "responses>=0.25.0",        # ADD - HTTP mocking
    "ruff>=0.4.0",
]
```

Add coverage threshold:
```toml
[tool.coverage.report]
fail_under = 80
show_missing = true
skip_covered = false

[tool.coverage.html]
directory = "apps/coverage_html"
```

Update pytest options:
```toml
[tool.pytest.ini_options]
testpaths = ["apps/tests"]
pythonpath = ["apps/src"]
python_files = ["test_*.py"]
python_functions = ["test_*"]
addopts = "-v --tb=short"
markers = [
    "slow: marks tests as slow",
    "integration: marks tests requiring external services",
]
```

### 0.2 Update .gitignore

Add:
```
# Coverage output
.coverage
coverage.xml
apps/coverage_html/
htmlcov/
```

### 0.3 Add Taskfile test commands

Add to Taskfile.yml:
```yaml
tasks:
  test:
    desc: Run all tests
    dir: "{{.CLAUDE_PATH}}"
    cmds:
      - uv run pytest

  test:fast:
    desc: Run tests, stop on first failure
    dir: "{{.CLAUDE_PATH}}"
    cmds:
      - uv run pytest -x --tb=short

  test:parallel:
    desc: Run tests in parallel
    dir: "{{.CLAUDE_PATH}}"
    cmds:
      - uv run pytest -n auto

  test:cov:
    desc: Run tests with coverage report
    dir: "{{.CLAUDE_PATH}}"
    cmds:
      - uv run pytest --cov --cov-report=term-missing

  test:verbose:
    desc: Run tests with verbose output
    dir: "{{.CLAUDE_PATH}}"
    cmds:
      - uv run pytest -vvs
```

### 0.4 Create conftest.py

`apps/tests/conftest.py`:
```python
"""Shared pytest fixtures for all tests."""
import pytest
from pathlib import Path

@pytest.fixture
def tmp_workspace(tmp_path: Path) -> Path:
    """Create a temporary workspace directory."""
    workspace = tmp_path / "workspace"
    workspace.mkdir()
    return workspace

@pytest.fixture
def mock_env(monkeypatch):
    """Factory for mocking environment variables."""
    def _mock(**env_vars):
        for key, value in env_vars.items():
            monkeypatch.setenv(key, value)
    return _mock
```

---

## Phase 1: shared/ Module (42 files) - Foundation

Priority: Highest - other modules depend on this.

### 1.1 aws_utils/core/ (schemas, session)
- Test Pydantic models validation
- Test boto3 session factory
- Mock AWS credentials

### 1.2 aws_utils/services/ (ec2, s3, sqs, sns, ses, organizations)
- Use moto for AWS service mocking
- Test discovery functions
- Test error handling

### 1.3 aws_utils/inventory/ (reader, writer)
- Test YAML read/write
- Test file path handling
- Test schema validation

**Target**: 42 files, ~200 test cases

---

## Phase 2: skills/ Module (56 files) - Core Features

### 2.1 git_manager/ (highest complexity)
- Mock git operations
- Test commit message generation
- Test identity detection

### 2.2 aws_login/, gcp_login/
- Mock SSO flows
- Test profile generation
- Test config validation

### 2.3 session_context/, project_metadata_builder/
- Test YAML parsing
- Test metadata collection
- Mock git repository

### 2.4 playwright_automation/
- Mock Playwright browser
- Test script generation
- Test error handling

### 2.5 gomplate_manager/, taskfile_manager/
- Test validation logic
- Test rule checking

**Target**: 56 files, ~350 test cases

---

## Phase 3: hooks/ Module (49 files) - Integration

### 3.1 Hook infrastructure
- Test stdin/stdout JSON protocol
- Test exit code handling

### 3.2 Individual hooks
- `plan_distributor`: Test file copy logic
- `session_context_injector`: Test context injection
- `rules_loader`: Test rule parsing
- `changelog_monitor`: Test version detection
- `submodule_auto_updater`: Mock git fetch/pull

**Target**: 49 files, ~250 test cases

---

## Phase 4: Integration Tests

### 4.1 End-to-end workflows
- Hook chain execution
- Skill invocation patterns
- AWS multi-account flows

### 4.2 Mark as `@pytest.mark.integration`
- Skip by default in fast runs
- Run in CI pipeline

---

## Coverage Milestones

| Phase | Files | Target Coverage | Cumulative |
|-------|-------|-----------------|------------|
| 0 | 0 | Infrastructure | 0% |
| 1 | 42 | 100% of shared/ | 29% |
| 2 | 56 | 100% of skills/ | 67% |
| 3 | 49 | 100% of hooks/ | 100% |
| 4 | - | Integration tests | 100%+ |

---

## Tasks

### Phase 0: Infrastructure
- [ ] Update pyproject.toml with new dependencies
- [ ] Update pyproject.toml with coverage threshold
- [ ] Update .gitignore with coverage output paths
- [ ] Add test tasks to Taskfile.yml
- [ ] Create apps/tests/conftest.py with shared fixtures

### Phase 1: shared/ tests
- [ ] Create apps/tests/shared/test_schemas.py
- [ ] Create apps/tests/shared/test_session.py
- [ ] Create apps/tests/shared/test_ec2.py (moto)
- [ ] Create apps/tests/shared/test_s3.py (moto)
- [ ] Create apps/tests/shared/test_organizations.py (moto)
- [ ] Create apps/tests/shared/test_inventory.py

### Phase 2-4: Deferred
- Implement after Phase 0-1 validated

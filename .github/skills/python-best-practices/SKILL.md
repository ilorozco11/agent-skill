---
name: python-best-practices
description: Python 2025 best practices for modern tooling, typing, linting/formatting, testing, dependency management, Pydantic v2, and secure development. Use when setting up or modernizing Python services, data pipelines, or libraries.
---

# Python Best Practices Skill

Use this skill to modernize Python projects with 2025 tooling conventions.

## Overview

Focus on consistent project structure, type safety, fast feedback loops, and secure dependency management.

## Guidelines

### Project layout and config
- Use `pyproject.toml` as the single source of truth for tooling (build system, lint, type checks).
- Favor `src/` layouts for libraries and keep top-level scripts minimal.
- Prefer Python 3.11+ and enable `from __future__ import annotations` for cleaner typing.

### Typing and validation
- Add type hints for public APIs and data boundaries; prefer explicit `TypedDict`/`Protocol` for interoperability.
- Run static checks with **mypy** or **pyright**; keep settings in `pyproject.toml`.
- Use Pydantic v2 patterns: `BaseModel`, `model_validate`, `model_dump`, and `field_validator`.

### Formatting and linting
- Standardize with **ruff** (`ruff check`) and **ruff format** (or Black if required by policy).
- Configure import sorting via Ruff (`I` rules) rather than separate isort configs.

### Dependency management
- Prefer **uv** for fast installs and locking, or **pip-tools** (`pip-compile`) for deterministic requirements.
- Pin dependencies and separate prod vs dev groups in `pyproject.toml`.

### Testing
- Use `pytest` with fixtures for reusable setup; keep unit tests fast and deterministic.
- Isolate I/O using temporary paths and monkeypatching.

### Data modeling
- Use `dataclasses` or **attrs** for lightweight models; keep mutable defaults safe.
- Use `pathlib.Path` for filesystem operations.

### Observability and logging
- Prefer structured logging with JSON output; include request IDs or job IDs.
- Avoid `print`; use `logging` with consistent log levels.

### Security
- Pin and audit dependencies (e.g., `pip-audit`, `uv pip audit`) in CI.
- Avoid `eval`, `pickle` for untrusted data, and keep secrets in env vars or secret managers.

## Examples

### pyproject.toml essentials
```toml
[project]
name = "example-service"
version = "0.1.0"
requires-python = ">=3.11"

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM", "RUF"]

[tool.ruff.format]
quote-style = "double"

[tool.pyright]
venvPath = "."
venv = ".venv"

[tool.mypy]
python_version = "3.11"
strict = true

[tool.pytest.ini_options]
addopts = "-q"
```

### Pydantic v2 model
```python
from pydantic import BaseModel, Field, field_validator

class UserEvent(BaseModel):
    user_id: str
    product_id: str
    event_type: str = Field(pattern=r"^(view|add_to_cart|purchase)$")

    @field_validator("user_id", "product_id")
    @classmethod
    def not_empty(cls, value: str) -> str:
        if not value.strip():
            raise ValueError("must not be empty")
        return value

payload = {"user_id": "u1", "product_id": "p1", "event_type": "view"}
validated = UserEvent.model_validate(payload)
```

### Pytest fixtures and pathlib
```python
from pathlib import Path
import pytest

@pytest.fixture
def temp_workspace(tmp_path: Path) -> Path:
    workspace = tmp_path / "workspace"
    workspace.mkdir()
    return workspace

def test_writes_file(temp_workspace: Path) -> None:
    target = temp_workspace / "output.txt"
    target.write_text("ok")
    assert target.read_text() == "ok"
```

### Structured logging
```python
import logging
import json

logger = logging.getLogger("service")
logger.setLevel(logging.INFO)

RESERVED_ATTRS = set(logging.LogRecord("", 0, "", 0, "", (), None).__dict__)

class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        payload = {
            "level": record.levelname,
            "message": record.getMessage(),
            "logger": record.name,
        }
        extra = {
            key: value
            for key, value in record.__dict__.items()
            if key not in RESERVED_ATTRS
        }
        if extra:
            payload["extra"] = extra
        return json.dumps(payload)

handler = logging.StreamHandler()
handler.setFormatter(JsonFormatter())
logger.addHandler(handler)

logger.info("processed event", extra={"event_id": "evt_123"})
```

### Dependency locking and audit
```bash
uv pip compile pyproject.toml -o requirements.txt
uv pip sync requirements.txt
pip-audit
```

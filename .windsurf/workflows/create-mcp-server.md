---
description: Create a new MCP server following AdoReviewLens conventions
---

# MCP Server Creation Workflow

**Important**: The directory where you invoke this workflow is considered the **root directory** for the new MCP server project.

## Phase 1: Planning & Research

### 1. Define the MCP Server Purpose
- Clearly identify what external service/API the MCP server will integrate with
- List the specific tools (functions) the server will expose
- Document required authentication/configuration (API keys, tokens, URLs)
- Identify the primary data models that will be returned

### 2. Research the Target API
// web
- Use web search to find official API documentation for the target service
- Identify authentication methods (OAuth, API keys, PAT tokens, etc.)
- Document rate limits, pagination, and error handling patterns
- Find Python SDK if available, or plan to use `requests` library
- Check for API versioning and stability guarantees

### 3. Plan the Project Structure
Based on AdoReviewLens template, plan these components:
- **Package name**: Use kebab-case for PyPI (e.g., `my-mcp-server`)
- **Module name**: Use snake_case for Python (e.g., `my_mcp_server`)
- **Tools to expose**: List each MCP tool with parameters
- **Data models**: Pydantic models for requests/responses
- **Configuration**: Environment variables needed

---

## Phase 2: Project Scaffolding

### 4. Create Directory Structure
Create the following structure in the **root directory**:
```
.
├── src/
│   └── {module_name}/
│       ├── __init__.py
│       ├── server.py
│       ├── cli.py
│       ├── api.py
│       ├── service.py
│       ├── models.py
│       ├── config.py
│       ├── errors.py
│       └── {client}.py
├── tests/
│   └── test_{core_logic}.py
├── scripts/
│   └── publish.sh
├── .env.example
├── .gitignore
├── pyproject.toml
├── server.json
├── README.md
└── Requirement.md
```

### 5. Create Core Configuration Files

**pyproject.toml** - Package metadata and dependencies:
```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "{package-name}"
version = "0.1.0"
description = "{Brief description of MCP server}"
authors = [{ name = "{Your Name}" }]
readme = "README.md"
requires-python = ">=3.10"
dependencies = [
    "requests>=2.31",
    "typer>=0.9",
    "pydantic>=2.0",
    "fastapi>=0.110",
    "uvicorn>=0.27",
    "mcp>=0.3.0",
    "python-dotenv>=1.0"
]

[project.scripts]
mcp = "{module_name}.cli:app"

[tool.setuptools.packages.find]
where = ["src"]
```

**server.json** - MCP server registry definition:
```json
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-07-09/server.schema.json",
  "name": "io.github.{username}/{package-name}",
  "description": "{Brief description}",
  "version": "0.1.0",
  "packages": [
    {
      "registry_type": "pypi",
      "identifier": "{package-name}",
      "version": "0.1.0"
    }
  ],
  "transport": {
    "type": "stdio",
    "command": ["python", "-m", "{module_name}.server"]
  },
  "metadata": {
    "homepage": "https://github.com/{username}/{repo-name}",
    "license": "MIT"
  }
}
```

**.gitignore**:
```
# Python artifacts
__pycache__/
*.py[cod]
*.egg-info/

# Virtual environments
.venv/

# IDE
.venv
.env

# Build artifacts
dist/
build/
*.spec

# Testing
.coverage
htmlcov/
.pytest_cache/
```

**.env.example** - Document all required environment variables:
```bash
# Required: {description}
SERVICE_API_URL=https://api.example.com
SERVICE_API_KEY=your-api-key-here

# Optional: {description}
SERVICE_DEFAULT_PARAM=default-value
```

---

## Phase 3: Core Implementation

### 6. Setup Quality Tools
Create `.pre-commit-config.yaml` in the root directory:
```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
  - repo: https://github.com/psf/black
    rev: 24.2.0
    hooks:
      - id: black
  - repo: https://github.com/pycqa/flake8
    rev: 7.0.0
    hooks:
      - id: flake8
```

Install hooks:
```bash
pip install pre-commit
pre-commit install
```

---

## Phase 3: Core Implementation

### 7. Implement Error Classes (`errors.py`)
Define custom exceptions following the pattern:
```python
"""Custom exceptions for the {ServerName} MCP."""

from __future__ import annotations


class MCPUserError(RuntimeError):
    """Exception raised for expected, user-facing errors."""

    def __init__(self, message: str, status: int) -> None:
        super().__init__(message)
        self.status = status


class {Service}RequestError(RuntimeError):
    """Raised when {Service} API returns an unexpected error."""

    def __init__(self, message: str, status: int) -> None:
        super().__init__(message)
        self.status = status


class MissingConfigurationError(RuntimeError):
    """Raised when required environment configuration is missing."""
```

### 8. Implement Data Models (`models.py`)
Use Pydantic with these conventions:
- Enable `populate_by_name=True` for camelCase/snake_case flexibility
- Use `Field(alias="camelCase")` for external API compatibility
- Create separate models for: Config, Request, Response, Error
- Use type hints with `Optional` for nullable fields

Example:
```python
"""Core data models for {ServerName} MCP."""

from __future__ import annotations

from typing import Optional
from pydantic import BaseModel, ConfigDict, Field


class MCPConfig(BaseModel):
    """Runtime configuration sourced from environment variables."""

    model_config = ConfigDict(populate_by_name=True)

    api_url: str
    api_key: str
    default_param: Optional[str] = None


class ResponseModel(BaseModel):
    """Normalized response representation returned by the MCP."""

    model_config = ConfigDict(populate_by_name=True)

    item_id: str = Field(alias="itemId")
    item_name: str = Field(alias="itemName")
    # Add more fields as needed
```

### 9. Implement Configuration Loader (`config.py`)
Load and validate environment variables:
```python
"""Configuration helpers for {ServerName} MCP."""

from __future__ import annotations

import os
from typing import Optional

from dotenv import load_dotenv

from .errors import MissingConfigurationError
from .models import MCPConfig


def load_config() -> MCPConfig:
    """Load configuration from environment variables."""

    load_dotenv()

    api_url = os.getenv("SERVICE_API_URL")
    api_key = os.getenv("SERVICE_API_KEY")

    if not api_url:
        raise MissingConfigurationError("SERVICE_API_URL is required")

    if not api_key:
        raise MissingConfigurationError("SERVICE_API_KEY is required")

    return MCPConfig(
        api_url=api_url.rstrip("/"),
        api_key=api_key,
        default_param=os.getenv("SERVICE_DEFAULT_PARAM"),
    )
```

### 10. Implement API Client (`{client}.py`)
Create a client class for the external service:
- Use context manager pattern (`__enter__`, `__exit__`)
- Centralize authentication headers
- Handle HTTP errors and convert to custom exceptions
- Implement retry logic if needed

Example:
```python
"""Client for {Service} API."""

from __future__ import annotations

from typing import Any, Dict
import requests

from .errors import {Service}RequestError
from .models import MCPConfig


class {Service}Client:
    """HTTP client for {Service} API."""

    def __init__(self, config: MCPConfig) -> None:
        self.config = config
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {config.api_key}",
            "Content-Type": "application/json",
        })

    def __enter__(self) -> "{Service}Client":
        return self

    def __exit__(self, *args: Any) -> None:
        self.session.close()

    def get_resource(self, resource_id: str) -> Dict[str, Any]:
        """Fetch a resource from the API."""
        url = f"{self.config.api_url}/resources/{resource_id}"
        response = self.session.get(url)

        if not response.ok:
            raise {Service}RequestError(
                f"Failed to fetch resource: {response.text}",
                status=response.status_code,
            )

        return response.json()
```

### 11. Implement Service Layer (`service.py`)
Business logic that orchestrates client calls and data transformation:
- Keep functions pure and testable
- Handle pagination if needed
- Transform external API responses to internal models
- Filter and normalize data

### 12. Implement MCP Server (`server.py`)
The main MCP entrypoint using FastMCP:
```python
"""Model Context Protocol server entrypoint."""

from __future__ import annotations

from typing import Optional

from mcp.server.fastmcp import FastMCP

from .errors import {Service}RequestError, MCPUserError, MissingConfigurationError
from .service import fetch_data

mcp = FastMCP("{ServerName}")


@mcp.tool()
def tool_name(
    param1: str,
    param2: Optional[int] = None,
) -> dict:
    """Tool description that will appear in MCP clients."""

    try:
        response = fetch_data(param1=param1, param2=param2)
        return response.model_dump(by_alias=True)
    except MissingConfigurationError as exc:
        raise ValueError(str(exc)) from exc
    except MCPUserError as exc:
        raise ValueError(f"{exc.status}: {exc}") from exc
    except {Service}RequestError as exc:
        raise RuntimeError(f"{Service} error ({exc.status}): {exc}") from exc


def main() -> None:
    """Run the MCP server using stdio transport."""
    mcp.run()


if __name__ == "__main__":
    main()
```

### 13. Implement CLI (`cli.py`)
Command-line interface using Typer:
- Mirror MCP tool parameters
- Output JSON for easy parsing
- Handle errors gracefully with proper exit codes

### 14. Implement HTTP API (`api.py`) [Optional]
FastAPI server for HTTP access:
- Create REST endpoints that mirror MCP tools
- Use Pydantic models for request/response validation
- Return proper HTTP status codes

---

## Phase 4: Testing & Documentation

### 15. Write Unit Tests
Create `tests/test_{core_logic}.py`:
- Test configuration loading with missing/invalid values
- Test data transformation logic
- Test error handling
- Use pytest fixtures for common test data
- Mock external API calls
- **Ensure >80% code coverage**

Run tests with coverage:
```bash
pip install pytest-cov
pytest --cov=src/{module_name} tests/
```

### 16. Create README.md
Document the following sections:
1. **Project description** - What the MCP server does
2. **Requirements** - Python version, API credentials needed
3. **Installation** - Local development setup with venv/uv
4. **CLI usage** - Example commands
5. **HTTP API server** - How to run (if applicable)
6. **MCP server** - How to run standalone
7. **MCP client registration** - Configuration examples for Claude Desktop, etc.
8. **Environment variables** - Complete reference

### 17. Create Requirement.md [Optional]
Document the original requirements, design decisions, and API research findings.

---

## Phase 5: Publishing

### 18. Create Publish Script (`scripts/publish.sh`)
Local publishing script for manual releases:
```bash
#!/usr/bin/env bash
set -euo pipefail

python -m build
python -m twine upload dist/*
```

### 19. Manual Publishing
1. Create PyPI account and project
2. Configure API token
3. Run publish script: `./scripts/publish.sh`

---

## Phase 6: Validation & Launch

### 20. Local Testing Checklist
- [ ] Virtual environment activates correctly
- [ ] `pip install -e .` succeeds
- [ ] CLI commands work with `.env` configuration
- [ ] MCP server starts: `python -m {module_name}.server`
- [ ] Unit tests pass: `pytest`
- [ ] **Test coverage is >80%**
- [ ] **Pre-commit hooks pass**: `pre-commit run --all-files`

### 21. MCP Client Integration Testing
Test with MCP Inspector:
```bash
uv run mcp dev src/{module_name}/server.py --with-editable .
```

Test with Claude Desktop or other MCP clients:
- Add server configuration
- Verify tools appear in client
- Test each tool with various inputs
- Verify error handling

### 22. Documentation Review
- [ ] README has clear installation instructions
- [ ] All environment variables documented
- [ ] Example usage commands provided
- [ ] MCP client configuration examples included
- [ ] API credentials acquisition explained

### 23. Pre-Release Checklist
- [ ] Version number updated in `pyproject.toml` and `server.json`
- [ ] CHANGELOG created (if applicable)
- [ ] All tests passing
- [ ] Published to TestPyPI successfully (optional)
- [ ] Tested installation from TestPyPI (optional)
- [ ] GitHub repository created with proper description

### 24. Release
- Tag version: `git tag v0.1.0`
- Push tag: `git push origin v0.1.0`
- Create GitHub release with notes

---

## Best Practices & Conventions

### Code Style
- Use `from __future__ import annotations` in all modules
- Type hint all function parameters and return values
- Use `Optional[T]` for nullable types
- Prefer `dict[str, Any]` over `Dict[str, Any]` (Python 3.10+)
- Use docstrings for all public functions and classes

### Error Handling
- Create custom exceptions for different error categories
- Always include status codes with API errors
- Convert external exceptions to internal exception types at boundaries
- Provide helpful error messages for users

### Configuration
- Use environment variables for all secrets and configuration
- Provide `.env.example` with documentation
- Validate all required configuration at startup
- Use `python-dotenv` for local development

### Data Models
- Use Pydantic for all data validation
- Enable `populate_by_name=True` for flexibility
- Use `Field(alias=...)` for external API compatibility
- Keep models in a dedicated `models.py` file

### Testing
- Write tests for core business logic
- Mock external API calls
- Test error conditions
- Use pytest fixtures for common setup
- **Maintain >80% coverage**

### Documentation
- Keep README concise but complete
- Provide copy-paste examples
- Document all environment variables
- Include troubleshooting section if needed

---

## Quick Reference: File Templates

### `__init__.py`
```python
"""Package initialization for {module_name}."""

from __future__ import annotations

__version__ = "0.1.0"
```

### Minimal `api.py`
```python
"""HTTP API server for {ServerName}."""

from __future__ import annotations

from fastapi import FastAPI, HTTPException

from .errors import MCPUserError, MissingConfigurationError, {Service}RequestError
from .models import ErrorResponse, FetchRequest
from .service import fetch_data

app = FastAPI(title="{ServerName} API")


@app.post("/api/v1/resource")
async def get_resource(request: FetchRequest) -> dict:
    """Fetch resource via HTTP API."""
    try:
        response = fetch_data(**request.model_dump())
        return response.model_dump(by_alias=True)
    except (MCPUserError, MissingConfigurationError) as exc:
        status = getattr(exc, "status", 400)
        raise HTTPException(status_code=status, detail=str(exc))
    except {Service}RequestError as exc:
        raise HTTPException(status_code=exc.status, detail=str(exc))
```

---

## Notes
- This workflow assumes Python 3.10+ and uses modern type hints
- All file paths are relative to the **root directory** where workflow is invoked
- Use web search liberally to research APIs and best practices
- Follow the existing **AdoTaskManager** patterns for consistency
- Adapt the structure as needed for your specific use case

---
description: MCP Server coding conventions and rules based on AdoReviewLens
---

# MCP Server Development Conventions

This document defines the coding standards, patterns, and rules to follow when creating MCP servers based on the AdoReviewLens template.

---

## Project Structure Rules

### Rule 1: Standard Directory Layout
**MUST** follow this exact structure:
```
{root}/
├── src/{module_name}/          # All source code
├── tests/                       # All test files
├── scripts/                     # Build/publish scripts
├── .env.example                 # Environment template
├── .gitignore                   # Git exclusions
├── pyproject.toml              # Package metadata
├── server.json                 # MCP registry definition
└── README.md                   # Documentation
```

### Rule 2: Module Organization
**MUST** include these core modules in `src/{module_name}/`:
- `__init__.py` - Package initialization with `__version__`
- `server.py` - MCP server entrypoint (FastMCP)
- `models.py` - Pydantic data models
- `config.py` - Configuration loading
- `errors.py` - Custom exception classes
- `service.py` - Business logic layer
- `{client}.py` - External API client
- `cli.py` - Command-line interface
- `api.py` - HTTP API (optional)

### Rule 3: Separation of Concerns
**MUST** maintain clear boundaries:
- **server.py**: MCP tool definitions only, delegate to service layer
- **service.py**: Business logic, orchestration, data transformation
- **{client}.py**: HTTP client, authentication, API calls
- **models.py**: Data structures only, no logic
- **config.py**: Environment loading and validation only
- **errors.py**: Exception definitions only

---

## Code Style Rules

### Rule 4: Future Annotations
**MUST** include in every `.py` file:
```python
from __future__ import annotations
```

### Rule 5: Type Hints
**MUST** type hint all:
- Function parameters
- Function return values
- Class attributes

**MUST** use modern syntax (Python 3.10+):
```python
# Good
def process(data: dict[str, Any]) -> list[str]:
    pass

# Bad
from typing import Dict, List
def process(data: Dict[str, Any]) -> List[str]:
    pass
```

### Rule 6: Optional Types
**MUST** use `Optional[T]` for nullable values:
```python
from typing import Optional

def fetch(url: str, timeout: Optional[int] = None) -> Optional[dict]:
    pass
```

### Rule 7: Docstrings
**MUST** include docstrings for:
- All public functions
- All classes
- All MCP tools (these appear in clients)

**Format**: Single-line for simple functions, multi-line for complex:
```python
def simple() -> str:
    """Return a greeting."""
    return "Hello"

def complex(param: str) -> dict:
    """
    Process the parameter and return results.

    Args:
        param: Input parameter to process

    Returns:
        Dictionary containing processed results
    """
    pass
```

---

## Pydantic Model Rules

### Rule 8: Model Configuration
**MUST** enable `populate_by_name=True` for all models:
```python
class MyModel(BaseModel):
    model_config = ConfigDict(populate_by_name=True)
```

### Rule 9: Field Aliases
**MUST** use `Field(alias="camelCase")` for external API compatibility:
```python
class ResponseModel(BaseModel):
    model_config = ConfigDict(populate_by_name=True)

    item_id: str = Field(alias="itemId")
    created_at: str = Field(alias="createdAt")
```

### Rule 10: Default Values
**MUST** provide defaults for optional fields:
```python
class MyModel(BaseModel):
    required_field: str
    optional_field: Optional[str] = None
    list_field: list[str] = Field(default_factory=list)
```

### Rule 11: Model Organization
**MUST** define models in this order in `models.py`:
1. Configuration models (`MCPConfig`)
2. Request models
3. Response models
4. Error models
5. Internal/helper models

---

## Error Handling Rules

### Rule 12: Custom Exceptions
**MUST** define these three exception types:
```python
class MCPUserError(RuntimeError):
    """User-facing errors (bad input, validation)."""
    def __init__(self, message: str, status: int) -> None:
        super().__init__(message)
        self.status = status

class {Service}RequestError(RuntimeError):
    """External API errors."""
    def __init__(self, message: str, status: int) -> None:
        super().__init__(message)
        self.status = status

class MissingConfigurationError(RuntimeError):
    """Missing required configuration."""
```

### Rule 13: Exception Conversion
**MUST** convert exceptions at boundaries:
```python
# In server.py
@mcp.tool()
def my_tool(param: str) -> dict:
    try:
        return service_function(param).model_dump(by_alias=True)
    except MissingConfigurationError as exc:
        raise ValueError(str(exc)) from exc
    except MCPUserError as exc:
        raise ValueError(f"{exc.status}: {exc}") from exc
    except ServiceRequestError as exc:
        raise RuntimeError(f"Service error ({exc.status}): {exc}") from exc
```

### Rule 14: Error Messages
**MUST** provide actionable error messages:
```python
# Good
raise MissingConfigurationError(
    "SERVICE_API_KEY is required. Set it in .env or environment."
)

# Bad
raise MissingConfigurationError("Missing config")
```

---

## Configuration Rules

### Rule 15: Environment Variables
**MUST** use environment variables for:
- API keys, tokens, passwords
- Service URLs
- Default values that vary by environment

**MUST NOT** hardcode secrets in source code.

### Rule 16: Configuration Loading
**MUST** implement `load_config()` function:
```python
def load_config() -> MCPConfig:
    """Load configuration from environment variables."""
    load_dotenv()

    api_key = os.getenv("SERVICE_API_KEY")
    if not api_key:
        raise MissingConfigurationError("SERVICE_API_KEY is required")

    return MCPConfig(api_key=api_key, ...)
```

### Rule 17: .env.example
**MUST** provide `.env.example` with:
- All required variables
- All optional variables
- Comments explaining each variable
- Example values (non-sensitive)

---

## API Client Rules

### Rule 18: Context Manager Pattern
**MUST** implement client as context manager:
```python
class ServiceClient:
    def __init__(self, config: MCPConfig) -> None:
        self.config = config
        self.session = requests.Session()

    def __enter__(self) -> "ServiceClient":
        return self

    def __exit__(self, *args: Any) -> None:
        self.session.close()
```

### Rule 19: Authentication Headers
**MUST** set authentication in session headers:
```python
self.session.headers.update({
    "Authorization": f"Bearer {config.api_key}",
    "Content-Type": "application/json",
})
```

### Rule 20: Error Handling in Client
**MUST** check response status and raise custom exceptions:
```python
def get_resource(self, resource_id: str) -> dict[str, Any]:
    response = self.session.get(f"{self.config.api_url}/resources/{resource_id}")

    if not response.ok:
        raise ServiceRequestError(
            f"Failed to fetch resource: {response.text}",
            status=response.status_code,
        )

    return response.json()
```

---

## MCP Server Rules

### Rule 21: FastMCP Usage
**MUST** use FastMCP for server implementation:
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("ServerName")

@mcp.tool()
def tool_name(param: str) -> dict:
    """Tool description."""
    pass

def main() -> None:
    """Run the MCP server using stdio transport."""
    mcp.run()

if __name__ == "__main__":
    main()
```

### Rule 22: Tool Return Values
**MUST** return dictionaries from MCP tools:
```python
@mcp.tool()
def fetch_data(id: str) -> dict:
    """Fetch data by ID."""
    response = service_function(id)
    return response.model_dump(by_alias=True)  # Convert Pydantic to dict
```

### Rule 23: Tool Parameters
**SHOULD** use `Optional` for optional parameters:
```python
@mcp.tool()
def search(
    query: str,
    limit: Optional[int] = None,
    offset: Optional[int] = None,
) -> dict:
    """Search with optional pagination."""
    pass
```

---

## Service Layer Rules

### Rule 24: Pure Functions
**SHOULD** make service functions pure and testable:
```python
def fetch_comments(
    *,  # Force keyword arguments
    pr_id: int | None = None,
    pr_url: str | None = None,
) -> CommentsResponse:
    """Fetch comments - pure business logic."""
    config = load_config()
    # ... logic here
    return CommentsResponse(...)
```

### Rule 25: Data Transformation
**MUST** transform external API data to internal models in service layer:
```python
def _normalize_comment(raw: dict[str, Any]) -> CommentModel:
    """Transform external API format to internal model."""
    return CommentModel(
        commentId=str(raw.get("id")),
        commentText=raw.get("text", ""),
        # ... more transformations
    )
```

---

## Testing Rules

### Rule 26: Test Organization
**MUST** create tests in `tests/` directory:
- Name test files `test_{module}.py`
- Test core business logic
- Test error conditions
- Use pytest fixtures

### Rule 27: Mock External Calls
**MUST** mock external API calls in tests:
```python
import pytest
from unittest.mock import patch

@patch('my_module.client.ServiceClient.get_resource')
def test_fetch_data(mock_get):
    mock_get.return_value = {"id": "123"}
    result = fetch_data("123")
    assert result.id == "123"
```

### Rule 28: Test Fixtures
**SHOULD** use fixtures for common test data:
```python
@pytest.fixture
def base_config() -> MCPConfig:
    return MCPConfig(
        api_url="https://api.example.com",
        api_key="test-key",
    )
```

### Rule 29: Test Coverage
**MUST** achieve high test coverage:
- **Goal**: >80% code coverage
- **Tools**: `pytest-cov`
- **Command**: `pytest --cov=src/{module_name} tests/`

---

## Quality Assurance Rules

### Rule 30: Pre-commit Hooks
**MUST** use `pre-commit` to ensure code quality before committing:
- **Formatter**: `black`
- **Linter**: `flake8`
- **Checks**: Trailing whitespace, end of file fixer, YAML check

**Configuration** (`.pre-commit-config.yaml`):
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

---

## Documentation Rules

### Rule 31: README Structure
**MUST** include these sections in README.md:
1. Project description
2. Requirements
3. Installation (local development)
4. CLI usage
5. HTTP API server (if applicable)
6. MCP server usage
7. MCP client registration examples
8. Environment variables reference

### Rule 32: Installation Instructions
**MUST** provide both venv and uv installation paths:
```markdown
### Local development

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -e .
```

Alternatively, if you prefer [uv](https://docs.astral.sh/uv/):

```bash
uv venv
source .venv/bin/activate
uv pip install -e .
```
```

### Rule 33: MCP Client Examples
**MUST** provide configuration examples for:
- Manual stdio configuration (JSON)
- MCP Inspector usage
- Claude Desktop or other popular clients

---

## Publishing Rules

### Rule 34: Version Tagging
**MUST** use semantic versioning with `v` prefix:
- Format: `v{major}.{minor}.{patch}`
- Example: `v0.1.0`, `v1.2.3`

---

## Package Metadata Rules

### Rule 35: pyproject.toml Structure
**MUST** include these sections:
```toml
[build-system]
[project]
[project.scripts]
[tool.setuptools.packages.find]
```

### Rule 36: Required Dependencies
**MUST** include these core dependencies:
- `mcp>=0.3.0` - MCP server framework
- `pydantic>=2.0` - Data validation
- `python-dotenv>=1.0` - Environment loading
- `requests>=2.31` - HTTP client (if needed)
- `typer>=0.9` - CLI framework
- `fastapi>=0.110` - HTTP API (optional)
- `uvicorn>=0.27` - ASGI server (optional)

### Rule 37: Python Version
**MUST** require Python 3.10 or higher:
```toml
requires-python = ">=3.10"
```

---

## Naming Conventions

### Rule 38: Package Names
- **PyPI package**: kebab-case (e.g., `ado-task-manager`)
- **Python module**: snake_case (e.g., `ado_task_manager`)
- **MCP server name**: PascalCase (e.g., `AdoTaskManager`)

### Rule 39: File Names
- **Modules**: snake_case (e.g., `service.py`, `azure_client.py`)
- **Tests**: `test_{module}.py` (e.g., `test_server.py`)
- **Scripts**: kebab-case (e.g., `publish.sh`)

### Rule 40: Function/Variable Names
- **Functions**: snake_case (e.g., `fetch_comments`, `load_config`)
- **Variables**: snake_case (e.g., `api_key`, `pr_id`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `DEFAULT_TIMEOUT`)
- **Private functions**: prefix with `_` (e.g., `_normalize_comment`)

---

## Security Rules

### Rule 41: Secrets Management
**MUST NOT**:
- Commit `.env` files
- Hardcode API keys, tokens, passwords
- Log sensitive information

**MUST**:
- Use environment variables for secrets
- Include `.env` in `.gitignore`
- Provide `.env.example` without real secrets

### Rule 42: Input Validation
**MUST** validate all user inputs:
- Use Pydantic models for validation
- Check for required fields
- Validate formats (URLs, IDs, etc.)
- Provide clear error messages

---

## Performance Rules

### Rule 43: Session Reuse
**MUST** reuse HTTP sessions:
```python
# Good - reuse session
with ServiceClient(config) as client:
    result1 = client.get_resource("1")
    result2 = client.get_resource("2")

# Bad - create new session each time
def get_resource(id: str) -> dict:
    response = requests.get(f"{url}/{id}")
    return response.json()
```

### Rule 44: Lazy Loading
**SHOULD** load configuration only when needed:
```python
# Good - load in function
def fetch_data() -> Response:
    config = load_config()
    # use config

# Avoid - load at module level (unless necessary)
config = load_config()  # Runs on import
```

---

## Git Rules

### Rule 45: .gitignore
**MUST** exclude:
```
__pycache__/
*.py[cod]
*.egg-info/
.venv/
.env
dist/
build/
.coverage
htmlcov/
.pytest_cache/
```

### Rule 46: Commit Messages
**SHOULD** use conventional commits:
- `feat:` - New features
- `fix:` - Bug fixes
- `docs:` - Documentation changes
- `test:` - Test additions/changes
- `chore:` - Build/tooling changes

---

## Quick Checklist

Before considering an MCP server complete, verify:

- [ ] All modules follow naming conventions
- [ ] All functions have type hints and docstrings
- [ ] All Pydantic models use `populate_by_name=True`
- [ ] Custom exceptions defined and used correctly
- [ ] Configuration loaded from environment variables
- [ ] `.env.example` provided with all variables
- [ ] HTTP client uses context manager pattern
- [ ] MCP tools return dictionaries
- [ ] Tests written for core logic
- [ ] **Test coverage is >80%**
- [ ] **Pre-commit hooks configured and passing**
- [ ] README includes all required sections
- [ ] `.gitignore` excludes sensitive files and build artifacts
- [ ] No secrets hardcoded in source
- [ ] Package metadata complete in `pyproject.toml`
- [ ] `server.json` properly configured

---

## Reference Implementation

For a complete reference implementation following all these rules, see:
- **AdoTaskManager**: `/home/om-surushe/Documents/AdoTaskManager`

Study the following files for patterns:
- `src/ado_task_manager/server.py` - MCP server structure
- `src/ado_task_manager/models.py` - Pydantic model patterns
- `src/ado_task_manager/errors.py` - Exception hierarchy
- `src/ado_task_manager/config.py` - Configuration loading
- `src/ado_task_manager/client.py` - API client pattern
- `src/ado_task_manager/service.py` - Business logic layer
- `src/ado_task_manager/cli.py` - CLI implementation
- `tests/test_server.py` - Testing patterns
- `.pre-commit-config.yaml` - Quality checks

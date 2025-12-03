# MCP Server Development Workflows

This directory contains workflows and conventions for building MCP servers based on the AdoReviewLens template.

## Available Workflows

### `/create-mcp-server` - Complete MCP Server Creation Guide
A comprehensive 24-step workflow that guides you through creating a production-ready MCP server from scratch.

**When to use**: Starting a new MCP server project

**Key features**:
- Phase-by-phase implementation guide
- Web search integration for API research
- Complete file templates and code examples
- Testing and CI/CD setup
- Publishing checklist

**Usage**: Invoke this workflow in any directory - that directory becomes the root for your new MCP server project.

### `mcp-conventions.md` - Coding Standards Reference
A detailed reference of 46 rules covering all aspects of MCP server development.

**When to use**: During development to ensure consistency

**Covers**:
- Project structure (Rules 1-3)
- Code style (Rules 4-7)
- Pydantic models (Rules 8-11)
- Error handling (Rules 12-14)
- Configuration (Rules 15-17)
- API clients (Rules 18-20)
- MCP servers (Rules 21-23)
- Service layer (Rules 24-25)
- Testing (Rules 26-28)
- Documentation (Rules 29-31)
- CI/CD (Rules 32-34)
- Package metadata (Rules 35-37)
- Naming conventions (Rules 38-40)
- Security (Rules 41-42)
- Performance (Rules 43-44)
- Git (Rules 45-46)

## Quick Start

To create a new MCP server:

1. Create a new directory for your project
2. Navigate to that directory
3. Use the `/create-mcp-server` workflow
4. Follow the 6 phases step-by-step
5. Reference `mcp-conventions.md` while coding

## Template Reference

The AdoReviewLens project serves as the reference implementation for all conventions:

**Key files to study**:
- `src/ado_review_lens/server.py` - MCP server structure
- `src/ado_review_lens/models.py` - Pydantic patterns
- `src/ado_review_lens/errors.py` - Exception hierarchy
- `src/ado_review_lens/config.py` - Configuration loading
- `src/ado_review_lens/azure.py` - API client pattern
- `src/ado_review_lens/service.py` - Business logic
- `tests/test_resolver.py` - Testing patterns
- `.github/workflows/ci.yml` - CI/CD setup

## Important Notes

- **Root Directory**: When using the workflow, the directory where you invoke it becomes the project root
- **Web Search**: The workflow includes `// web` annotations where you should use web search to research APIs
- **Conventions**: All code should follow the rules in `mcp-conventions.md`
- **Type Safety**: Python 3.10+ required for modern type hints
- **Dependencies**: Core dependencies include `mcp`, `pydantic`, `requests`, `typer`, `python-dotenv`

## Workflow Phases Overview

1. **Planning & Research** - Define purpose, research APIs, plan structure
2. **Project Scaffolding** - Create directories and configuration files
3. **Core Implementation** - Build errors, models, config, client, service, server
4. **Testing & Documentation** - Write tests, README, requirements
5. **CI/CD & Publishing** - Set up GitHub Actions, PyPI publishing
6. **Validation & Launch** - Test locally, integrate with MCP clients, release

## Support

For questions or issues with the workflow:
- Review the reference implementation in this repository
- Check the conventions document for specific rules
- Consult the MCP documentation at https://modelcontextprotocol.io/

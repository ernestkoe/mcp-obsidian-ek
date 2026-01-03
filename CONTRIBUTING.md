# Contributing

Thanks for your interest in contributing to mcp-obsidian-ek!

## Development Setup

### Prerequisites

- Python 3.11+
- [uv](https://docs.astral.sh/uv/) package manager
- Obsidian with [Local REST API](https://github.com/coddingtonbear/obsidian-local-rest-api) plugin

### Getting Started

```bash
# Clone the repo
git clone https://github.com/ernestkoe/mcp-obsidian-ek.git
cd mcp-obsidian-ek

# Install dependencies
uv sync

# Run the server locally
uv run mcp-obsidian-ek
```

### Local MCP Configuration

To test with Claude Desktop or Claude Code, add to your MCP config:

```json
{
  "mcpServers": {
    "obsidian-dev": {
      "command": "uv",
      "args": ["--directory", "/path/to/mcp-obsidian-ek", "run", "mcp-obsidian-ek"],
      "env": {
        "OBSIDIAN_API_KEY": "<your_api_key>"
      }
    }
  }
}
```

## Commands

```bash
# Install/sync dependencies
uv sync

# Run server
uv run mcp-obsidian-ek

# Run all tests
uv run pytest

# Run specific test file
uv run pytest tests/test_obsidian.py -v

# Run specific test
uv run pytest tests/test_obsidian.py::TestObsidianClient::test_get_file_contents -v

# Type checking
uv run pyright src/

# Linting
uv run ruff check src/
```

## Architecture

The codebase follows a simple three-layer structure:

```
src/mcp_obsidian/
├── __init__.py    # Package entry point
├── server.py      # MCP server setup and request routing
├── tools.py       # Tool handler classes
└── obsidian.py    # HTTP client for Obsidian REST API
```

### Layer Responsibilities

1. **`server.py`** - MCP server setup and request routing. Registers tool handlers and dispatches incoming tool calls. Entry point is `main()` which runs the stdio server.

2. **`tools.py`** - Tool handler classes. Each tool extends `ToolHandler` base class with:
   - `get_tool_description()` - Returns MCP `Tool` schema
   - `run_tool(args)` - Executes the tool logic

3. **`obsidian.py`** - HTTP client for Obsidian's Local REST API. The `Obsidian` class wraps all REST endpoints with `_safe_call()` for error handling.

## Adding a New Tool

1. **Add the API method** in `obsidian.py` if needed:
   ```python
   def my_new_endpoint(self, param: str) -> dict[str, Any]:
       return self._safe_call("GET", f"/some/endpoint/{param}")
   ```

2. **Create a tool handler** in `tools.py`:
   ```python
   class MyNewToolHandler(ToolHandler):
       def get_tool_description(self) -> Tool:
           return Tool(
               name="obsidian_my_new_tool",
               description="What this tool does",
               inputSchema={
                   "type": "object",
                   "properties": {
                       "param": {
                           "type": "string",
                           "description": "Parameter description"
                       }
                   },
                   "required": ["param"]
               }
           )

       async def run_tool(self, args: dict[str, Any]) -> Sequence[TextContent]:
           result = self.obsidian.my_new_endpoint(args["param"])
           return [TextContent(type="text", text=json.dumps(result, indent=2))]
   ```

3. **Register the handler** in `server.py`:
   ```python
   add_tool_handler(MyNewToolHandler(obsidian))
   ```

4. **Add tests** in `tests/`

## Testing

Tests use pytest with mocked HTTP responses (no live Obsidian needed):

```bash
# Run all tests
uv run pytest

# Run with coverage
uv run pytest --cov=src/mcp_obsidian

# Run specific test
uv run pytest tests/test_tools.py::test_my_new_tool -v
```

## Debugging

### MCP Inspector

The [MCP Inspector](https://github.com/modelcontextprotocol/inspector) provides a web UI for testing tools:

```bash
npx @modelcontextprotocol/inspector uv --directory /path/to/mcp-obsidian-ek run mcp-obsidian-ek
```

### Server Logs

On macOS, Claude Desktop logs MCP server output to:

```bash
tail -f ~/Library/Logs/Claude/mcp-server-obsidian-dev.log
```

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `OBSIDIAN_API_KEY` | Yes | — | API key from Local REST API plugin |
| `OBSIDIAN_HOST` | No | `127.0.0.1` | Obsidian host address |
| `OBSIDIAN_PORT` | No | `27124` | Obsidian REST API port |
| `OBSIDIAN_PROTOCOL` | No | `https` | Protocol (https/http) |

## Pull Requests

1. Fork the repo and create a feature branch
2. Make your changes with tests
3. Ensure `uv run pytest` and `uv run pyright src/` pass
4. Use [conventional commits](https://www.conventionalcommits.org/) for commit messages:
   - `feat:` - New features (triggers minor version bump)
   - `fix:` - Bug fixes (triggers patch version bump)
   - `docs:` - Documentation only
   - `refactor:` - Code changes that don't add features or fix bugs
5. Open a PR against `main`

## Release Process

Releases are automated via GitHub Actions:

1. Push to `main` triggers CI (tests + type checking)
2. [python-semantic-release](https://python-semantic-release.readthedocs.io/) analyzes commits
3. If `feat:` or `fix:` commits found, it bumps version and creates a GitHub release
4. Package is automatically published to PyPI

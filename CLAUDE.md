# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MCP server to interact with Obsidian via the Local REST API community plugin. Written in Python, uses the Model Context Protocol (MCP) to expose Obsidian vault operations as tools.

## Development Commands

### Environment Setup
```bash
uv sync  # Sync dependencies and update lockfile
```

### Running the Server
```bash
# Development/local testing
uv --directory /path/to/mcp-obsidian run mcp-obsidian

# Published version (via uvx)
uvx mcp-obsidian
```

### Debugging
```bash
# Use MCP Inspector for interactive debugging
npx @modelcontextprotocol/inspector uv --directory /path/to/mcp-obsidian run mcp-obsidian

# Watch server logs (macOS)
tail -n 20 -f ~/Library/Logs/Claude/mcp-server-mcp-obsidian.log
```

### Type Checking
```bash
uv run pyright  # Run type checking (pyright is in dev dependencies)
```

## Architecture

### Core Components

**server.py** - MCP server setup and tool registration
- Initializes MCP server with name "mcp-obsidian"
- Loads environment variables (OBSIDIAN_API_KEY required, OBSIDIAN_HOST/PORT optional)
- Registers all tool handlers via `add_tool_handler()`
- Routes tool calls to appropriate handlers via `get_tool_handler()`
- Implements MCP `list_tools()` and `call_tool()` handlers

**obsidian.py** - Obsidian REST API client
- Single `Obsidian` class handles all HTTP communication with Obsidian Local REST API plugin
- Configuration via env vars: OBSIDIAN_PROTOCOL (default: https), OBSIDIAN_HOST (default: 127.0.0.1), OBSIDIAN_PORT (default: 27124)
- SSL verification disabled by default (`verify_ssl=False`) for local connections
- All API methods use `_safe_call()` wrapper for consistent error handling
- Timeout set to (3, 6) seconds for connect/read

**tools.py** - Tool handler implementations
- Base class `ToolHandler` defines interface: `get_tool_description()` and `run_tool()`
- Each tool is a separate class inheriting from `ToolHandler`
- Tool handlers instantiate their own `Obsidian` client on each call
- Returns MCP-compatible `TextContent | ImageContent | EmbeddedResource` sequences

### Tool Handler Pattern

To add a new tool:
1. Create new class inheriting from `ToolHandler` in `tools.py`
2. Implement `get_tool_description()` returning MCP `Tool` schema
3. Implement `run_tool(args: dict)` for tool logic
4. Register in `server.py` via `add_tool_handler(YourToolHandler())`

### Available Tools

- `obsidian_list_files_in_vault` - List root directory files
- `obsidian_list_files_in_dir` - List specific directory files
- `obsidian_get_file_contents` - Get single file content
- `obsidian_batch_get_file_contents` - Get multiple files with headers
- `obsidian_simple_search` - Text search across vault
- `obsidian_complex_search` - JsonLogic query search (supports glob/regexp)
- `obsidian_patch_content` - Insert relative to heading/block/frontmatter
- `obsidian_append_content` - Append to file
- `obsidian_put_content` - Create/replace file
- `obsidian_delete_file` - Delete file/directory (requires confirmation)
- `obsidian_get_periodic_note` - Get current periodic note (daily/weekly/monthly/quarterly/yearly)
- `obsidian_get_recent_periodic_notes` - Get recent periodic notes
- `obsidian_get_recent_changes` - Get recently modified files via DQL query

### Environment Configuration

Required:
- `OBSIDIAN_API_KEY` - API key from Obsidian Local REST API plugin

Optional:
- `OBSIDIAN_HOST` - Default: 127.0.0.1
- `OBSIDIAN_PORT` - Default: 27124
- `OBSIDIAN_PROTOCOL` - Default: https (also accepts http)

Can be configured via `.env` file or MCP server config (preferred).

## Key Design Decisions

- **Tool isolation**: Each tool handler creates its own Obsidian client instance rather than sharing a global client
- **Error handling**: All API calls wrapped in `_safe_call()` to convert HTTP errors to user-friendly messages
- **MCP protocol**: Tools return sequences of MCP content types (TextContent, ImageContent, EmbeddedResource)
- **Type safety**: Uses Python type hints throughout, pyright for type checking
- **Entry point**: `main()` function in `__init__.py` registered as console script in `pyproject.toml`

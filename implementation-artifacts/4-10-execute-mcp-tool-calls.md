# Story 4-10: Execute MCP Tool Calls (Stdio Driver)

## Overview
**Epic**: 4 – Execution Core
**Priority**: High
**Status**: `ready-for-design`

## Goal
Enable the Runtime to execute tools provided by Model Context Protocol (MCP) servers using the `stdio` transport. This allows the Agent to interact with external systems (filesystem, git, databases) via standard MCP servers.

## Business Value
- **Extensibility**: Instantly unlocks a vast ecosystem of existing MCP servers (e.g., brave-search, filesystem, git).
- **Standardization**: Replaces ad-hoc tool implementations with a standard protocol.

## Acceptance Criteria

### 1. MCP Client Integration
- **Given** a visible MCP server configuration (command + args)
- **When** the Runtime initializes
- **Then** it should establish a connection to the MCP server via stdio.

### 2. Tool Discovery
- **Given** a connected MCP server
- **When** the Agent asks for available tools
- **Then** the Runtime should list tools exposed by the MCP server (mapped to `function` definitions for the LLM).

### 3. Tool Execution
- **Given** the LLM decides to call an MCP tool
- **When** the Runtime receives the tool call
- **Then** it should:
    1. Translate the call to an MCP `CallToolRequest`.
    2. Send it to the server.
    3. Receive the `CallToolResult`.
    4. Return the result to the LLM.

### 4. Error Handling
- **Given** an MCP tool call fails (stderr output or protocol error)
- **When** the error occurs
- **Then** the Runtime should catch the error and return it as a tool output to the LLM (not crash the runtime).

## Technical Components / Changes
1.  **`McpClientService`**: Wrapper around the official MCP SDK (or custom implementation) to manage stdio child processes.
2.  **`ToolRegistry` Extension**: Update existing tool registry to support "dynamic" tools from MCP clients.
3.  **`ExecutionEngine` Update**: Logic to route tool calls starting with `mcp:` (or similar namespace) to the correct client.
4.  **Config**: Basic configuration to specific which MCP servers to launch (initially hardcoded or read from `bmad.json`/settings).

## Dependencies
- Story 4-6: Execute Tool Calls (Sandboxed Filesystem) - *Foundation for tool execution*
- Story 5-5: Settings Page - *For configuring MCP servers (future)*

## Verification Plan

### Manual Verification
1.  **Setup**: Configure a simple MCP server (e.g., `mac-filesystem-server` or a specialized echo server).
2.  **Discovery**: Start a chat/workflow. Check logs to see MCP tools listed in the system prompt.
3.  **Execution**: Ask the agent to use the tool (e.g., "List files in directory").
4.  **Result**: Verify the agent sees the real output.

### Automated Tests
- Unit tests for `McpClientService` (mocking the stdio transport).
- Integration test: Spawn a dummy node process acting as an MCP server and perform a Request/Response cycle.

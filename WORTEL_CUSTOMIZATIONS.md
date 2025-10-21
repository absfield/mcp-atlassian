# Absfield Customizations to mcp-atlassian
## What We Did

We forked the official `sooperset/mcp-atlassian` repository and built a custom Docker image to use with Wortel instead of relying on Docker Desktop's unstable MCP toolkit.

## Changes Made

### 1. Docker Image Build
**What**: Built a custom Docker image from the existing Dockerfile
**Why**: To run the MCP server independently without Docker Desktop's MCP toolkit
**How**:
```bash
docker build -t wortel/mcp-atlassian:latest .
docker tag wortel/mcp-atlassian:latest wortel/mcp-atlassian:v1.0.0
```

### 2. No Code Changes Required
**Important**: We didn't need to modify any Python code in the mcp-atlassian server itself!

The existing server already supported:
- ✅ Stdio transport (standard input/output communication)
- ✅ Environment variable configuration for credentials
- ✅ MCP protocol version 2025-03-26
- ✅ Jira and Confluence integration

### 3. Integration Method
Instead of using Docker Desktop's `docker mcp` commands (which broke in version 4.48.0), we:

**Run the server directly:**
```bash
docker run --rm -i \
  -e JIRA_URL=https://your-domain.atlassian.net/ \
  -e JIRA_USERNAME=your-email@example.com \
  -e JIRA_API_TOKEN=your-token \
  -e CONFLUENCE_URL=https://your-domain.atlassian.net/ \
  -e CONFLUENCE_USERNAME=your-email@example.com \
  -e CONFLUENCE_API_TOKEN=your-token \
  wortel/mcp-atlassian:latest
```

**Communicate via stdio**: Send JSON-RPC requests to stdin, receive responses from stdout

## Wortel Backend Changes

The changes were all made in the Wortel backend to communicate with this server:

### File: `backend/services/atlassian_mcp_stdio.py`
**What it does**: Python client that communicates with the MCP server via stdio

**Key protocol requirements we implemented:**
1. **Correct protocol version**: `"2025-03-26"` (not `"0.1.0"`)
2. **Three-step handshake**:
   - Send `initialize` request with capabilities
   - Receive server response with its capabilities
   - Send `notifications/initialized` notification to complete setup
3. **Proper JSON-RPC format**: Each message on a new line, with correct fields

### File: `backend/services/atlassian_mcp_service.py`
**What changed**: Instead of calling `docker mcp tools call`, we:
- Start the Docker container ourselves with credentials
- Use the stdio client to send requests
- Parse MCP responses into Wortel's data format

## Why This Approach?

### Problems with Docker Desktop MCP Toolkit:
- ❌ Broke in version 4.48.0 (removed "obsolete mcp key")
- ❌ Secret management was unreliable
- ❌ Gateway had bugs (nil pointer crashes)
- ❌ Not suitable for production deployment

### Benefits of Our Custom Solution:
- ✅ Full control over the MCP server
- ✅ No dependency on Docker Desktop's experimental features
- ✅ Works reliably across Docker Desktop versions
- ✅ Can be deployed to production (clients can use same image)
- ✅ Follows MCP best practices with proper protocol implementation

## Testing

**Test script**: `wortel/test_custom_mcp.py`

Verifies:
- Docker image starts correctly
- MCP protocol handshake succeeds
- Jira search returns results
- Confluence search works (if configured)
- Clean shutdown

## Future Customizations

Since we control this fork, we can:
- Add custom Jira/Confluence workflows
- Implement caching for frequently accessed data
- Add Wortel-specific preprocessing
- Customize authentication methods
- Add monitoring/logging for production

## Summary

**What we customized**: Nothing in mcp-atlassian code!
**What we built**: A Docker image + Python stdio client
**What we achieved**: Reliable, production-ready Atlassian MCP integration
**Time saved**: Avoided fighting with Docker Desktop's broken MCP toolkit

The beauty of this solution is its simplicity - we used the mcp-atlassian server exactly as designed, just changed *how* we run and communicate with it.

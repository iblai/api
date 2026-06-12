# iblai-agent-chat

MCP (Model Context Protocol) server for IBL.ai mentor chat interactions, enabling AI assistants to communicate with configured AI mentors.

**Base URL:** `https://asgi.data.iblai.app` (or your IBL.ai deployment)

**Type:** Hosted (no local installation required)

## Overview

The iblai-agent-chat server provides a bridge between MCP-compatible AI assistants (like Claude) and IBL.ai's mentor system. This allows users to interact with specialized AI mentors configured for specific domains, courses, or purposes through their preferred AI assistant interface.

## Configuration

### Authentication

The server uses Api-Token authentication via the `Authorization` header, plus a mentor identifier. You need:
1. A Platform API Key from your IBL.ai admin panel
2. The mentor's unique ID via the `X-Mentor-Unique-Id` header

### Required Headers

| Header | Description | Required |
|--------|-------------|----------|
| `Authorization` | Api-Token for authentication | Yes |
| `X-Mentor-Unique-Id` | Unique identifier of the mentor to interact with | Yes |

## Usage

### Connect from Claude Desktop / Cursor

Add this to your configuration file:

- **Claude Desktop (macOS)**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Claude Desktop (Windows)**: `%APPDATA%\Claude\claude_desktop_config.json`
- **Cursor**: Settings > Features > MCP Servers

```json
{
  "mcpServers": {
    "iblai-agent-chat": {
      "transport": "streamable-http",
      "url": "https://asgi.data.iblai.app/mcp/agent-chat/",
      "headers": {
        "Authorization": "Api-Token YOUR_API_TOKEN",
        "X-Mentor-Unique-Id": "YOUR_MENTOR_UNIQUE_ID"
      }
    }
  }
}
```

### Connect from Claude Code

```bash
claude mcp add iblai-agent-chat --transport http https://asgi.data.iblai.app/mcp/agent-chat/ --header "Authorization: Api-Token YOUR_API_TOKEN" --header "X-Mentor-Unique-Id: YOUR_MENTOR_UNIQUE_ID"
```

## Available Tools

### `get_mentor_response`

Send a prompt to the configured mentor and receive a response.

**Parameters:**
- `prompt`: The message or question to send to the mentor (required)

**Returns:** The mentor's response as a string

**Example Usage:**

```
User: "What are the key concepts in machine learning?"
Tool Call: get_mentor_response(prompt="What are the key concepts in machine learning?")
Response: "Machine learning encompasses several key concepts..."
```

---

## How It Works

1. **Authentication**: The server validates the user's Bearer token against IBL.ai's authentication system
2. **Mentor Resolution**: The `X-Mentor-Unique-Id` header identifies which mentor to route the conversation to
3. **LLM Invocation**: The prompt is processed through IBL.ai's LLM service with the mentor's configured context, system prompts, and knowledge base
4. **Response**: The mentor's response is returned through the MCP tool

## Mentor Configuration

Mentors in IBL.ai can be configured with:
- **System prompts**: Define the mentor's personality and behavior
- **Knowledge bases**: Connect documents, courses, or custom data
- **LLM models**: Choose the underlying language model
- **Guardrails**: Set content policies and response filters
- **Tools**: Enable specific capabilities like code execution or web search

## Use Cases

### Educational Assistance

Connect to a course-specific mentor for tutoring:

> "Can you explain the concept of recursion with a simple example?"

### Customer Support

Route queries to a product knowledge mentor:

> "How do I configure the integration settings?"

### Domain Expertise

Interact with specialized mentors:

> "What are the compliance requirements for GDPR?"

### Multi-Mentor Workflows

Configure multiple mentor connections for different tasks:

```json
{
  "mcpServers": {
    "python-tutor": {
      "transport": "streamable-http",
      "url": "https://asgi.data.iblai.app/mcp/agent-chat/",
      "headers": {
        "Authorization": "Api-Token YOUR_API_TOKEN",
        "X-Mentor-Unique-Id": "PYTHON_MENTOR_ID"
      }
    },
    "data-science-expert": {
      "transport": "streamable-http",
      "url": "https://asgi.data.iblai.app/mcp/agent-chat/",
      "headers": {
        "Authorization": "Api-Token YOUR_API_TOKEN",
        "X-Mentor-Unique-Id": "DS_MENTOR_ID"
      }
    }
  }
}
```

## Finding Your Mentor ID

To find a mentor's unique ID:

1. **Via IBL.ai Dashboard**: Navigate to Mentors → Select Mentor → Copy the Unique ID from settings
2. **Via API**: Use the mentor list endpoint: `GET /api/ai-mentor/orgs/{org}/users/{user_id}/mentors/`
3. **Via iblai-search**: Use the mentor search tools to discover mentors and their IDs

## Error Handling

| Error | Description | Resolution |
|-------|-------------|------------|
| "Could not authenticate mentor via X-Mentor-Unique-Id" | Invalid or missing mentor ID | Verify the mentor ID in the header |
| 401 Unauthorized | Invalid Bearer token | Re-authenticate with valid credentials |
| 403 Forbidden | User lacks access to this mentor | Check mentor permissions |
| 500 Server Error | Internal processing error | Contact support |

## Best Practices

1. **Use Specific Mentors**: Configure separate mentor connections for different use cases
2. **Handle Errors Gracefully**: Implement fallback logic for authentication failures
3. **Monitor Usage**: Track mentor interactions via the analytics MCP
4. **Secure Credentials**: Store Bearer tokens and mentor IDs securely

## Rate Limiting

The mentor chat service may be subject to rate limiting based on:
- User tier
- Organization quotas
- LLM provider limits

Check with your IBL.ai administrator for specific limits.

## Related MCP Servers

| Server | Description |
|--------|-------------|
| [iblai-analytics](../iblai-analytics) | Monitor mentor usage and costs |
| [iblai-search](../iblai-search) | Discover available mentors |
| [iblai-agent-create](../iblai-agent-create) | Create and manage AI mentors |

## License

Proprietary - IBL.ai


# BuildBear MCP Server

This MCP server allows supported harnesses and LLMs to use the BuildBear API via tool calls. The documented [Sandbox API](https://www.buildbear.io/docs/api-reference/sandbox-api) and [Explorer API](https://www.buildbear.io/docs/api-reference/explorer-api) are implemented.

## Setup

To test out the MCP server with Claude desktop:

1. Add your BuildBear API key as `BB_API_KEY` in your MCP client's environment variables.
2. Run `npm run build`.
3. [Configure Claude desktop as instructed here.](https://modelcontextprotocol.io/quickstart/server#testing-your-server-with-claude-for-desktop)

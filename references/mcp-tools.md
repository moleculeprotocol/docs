---
icon: toolbox
---

# MCP Tools

## Molecule MCP Server Documentation

### Overview

The Molecule MCP (Model Context Protocol) server enables AI assistants to access DeSci ecosystem data through natural language. Users can query AI assistants like Claude with questions like "What IPTs are available?" to receive real-time responses.

#### What is MCP?

Model Context Protocol is an open standard that allows AI assistants to utilize external tools, enabling access to current data from sources like Molecule's datasets.

### MCP Server Functionality

The MCP server bridges AI assistants and Molecule Protocol data. It allows AI to fetch real data about IPTs, prices, and project activity, rather than just relying on pre-trained knowledge.

#### Example Interaction

* **Query**: "What's the price history for HAIR?"
* **Process**:
  1. AI Assistant interprets the request.
  2. Selects the appropriate tool.
  3. Calls the Molecule MCP server at `https://molecule-mcp.vercel.app/mcp`.

#### Data Sources

The MCP server pulls information from various resources like:

* Molecule API (GraphQL)
* Sanity CMS (Categories)
* GeckoTerminal (Price Data)

### Quick Start Guide

#### Claude Desktop Integration

To enable Molecule tools in Claude Desktop:

* **macOS**: Add to `~/Library/Application Support/Claude/claude_desktop_config.json`
* **Windows**: Add to `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "molecule": {
      "url": "https://molecule-mcp.vercel.app/mcp"
    }
  }
}
```

Restart Claude Desktop after saving.

#### Claude Code Integration

For CLI using Claude Code, add `.mcp.json` to your project root:

```json
{
  "mcpServers": {
    "molecule": {
      "type": "url",
      "url": "https://molecule-mcp.vercel.app/mcp"
    }
  }
}
```

#### Other MCP Clients

Connect using Streamable HTTP transport:

{% code overflow="wrap" %}
```javascript
import { experimental_createMCPClient as createMCPClient } from 'ai';
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js';

const client = await createMCPClient({
  transport: new StreamableHTTPClientTransport(
    new URL('https://molecule-mcp.vercel.app/mcp')
  )
});
const tools = await client.tools();
```
{% endcode %}

### Available Tools

The MCP server exposes five main tools:

1. **molecule-get-ipts**: Retrieves Intellectual Property Tokens with detailed market data.
2. **molecule-get-project-activity**: Fetches recent activity for a specified IPT.
3. **molecule-get-ipt-categories**: Lists IPT categories with statistics.
4. **molecule-get-project-summary**: Provides a comprehensive analysis of a specific IPT.
5. **molecule-get-ipt-historic-prices**: Gets historical OHLCV price data for trends analysis.

#### Tool Details

**1. molecule-get-ipts**

* Retrieves token metadata and market information.
* **Prompts**: "What IPTs are available in the Molecule ecosystem?"

**2. molecule-get-project-activity**

* Fetches recent activity for a specific IPT.
* **Parameter**: `iptSymbol` (string, required)
* **Prompts**: "What's the recent activity for VITA-FAST?"

**3. molecule-get-ipt-categories**

* Lists categories with aggregated data.
* **Prompts**: "What research categories does Molecule cover?"

**4. molecule-get-project-summary**

* Provides a detailed overview of a project.
* **Parameter**: `iptSymbol` (string, required)
* **Prompts**: "Give me a full summary of the VITA-FAST project."

**5. molecule-get-ipt-historic-prices**

* Retrieves historical price data.
* **Parameters**:
  * `iptSymbol` (string, required)
  * `days` (number, required)
* **Prompts**: "Show me the 30-day price history for HAIR."

#### Example Conversations

* **Ecosystem Overview**: "What's happening in the Molecule ecosystem?"
* **Project Deep Dive**: "I'm interested in the HAIR project. What can you tell me?"
* **Price Analysis**: "How has VITA-FAST performed over the last month?"

### Self-Hosting

For private deployments or custom configurations:

#### Requirements

* Node.js 18 or higher
* Vercel account or any Node.js platform
* Molecule API key

#### Deploy Steps

1.  Clone the repository:

    ```shell
    git clone https://github.com/moleculeprotocol/molecule-mcp
    cd molecule-mcp
    ```
2.  Install dependencies:

    ```shell
    pnpm install
    ```
3.  Deploy to Vercel:

    ```shell
    vercel --prod
    ```

Add environment variables in Vercel under Settings.

#### Local Development

*   Start with:

    ```shell
    vercel dev
    ```
*   Test with:

    ```shell
    node scripts/test-client.mjs http://localhost:3000/mcp
    ```

### Caching

The server uses Redis to cache data, enhancing performance and abiding by upstream limits.

#### Cache Durations

* IPT list: 10 minutes
* Project activities: 10 minutes
* Categories: 10 minutes
* Price history: 5 minutes

### Rate Limits

* **Public Endpoint**: 100 requests/minute
* **Self-hosted**: Unlimited

### Programmatic Integration

For AI applications requiring Molecule data, use the `@moleculexyz/ai` package.

#### Installation

```shell
npm install @moleculexyz/ai
```

#### Usage with Vercel AI SDK

```javascript
import { createToolRegistry } from '@moleculexyz/ai';
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';

const registry = await createToolRegistry({
  mcpUrl: 'https://molecule-mcp.vercel.app/mcp',
  toolSet: 'all'
});

const result = await generateText({
  model: openai('gpt-4o'),
  tools: registry.tools,
  prompt: 'What IPTs are available in the longevity category?'
});
```

### Troubleshooting

* Ensure configuration file syntax is valid.
* First request may be slower; consider using Redis for caching.
* Deploy your own instance if facing rate limit errors.

### Resources

* **Public Endpoint**: [https://molecule-mcp.vercel.app/mcp](https://molecule-mcp.vercel.app/mcp)
* **Source Code**: [GitHub](https://github.com/moleculeprotocol/molecule-mcp)
* **MCP Specification**: [https://modelcontextprotocol.io](https://modelcontextprotocol.io)
* **API Key Request**: Contact the Molecule team for deployment.

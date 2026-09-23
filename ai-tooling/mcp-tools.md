---
icon: toolbox
---

# MCP Tools

## Molecule MCP Server Documentation

### Overview

The Molecule MCP (Model Context Protocol) server enables AI assistants to access DeSci ecosystem data through natural language. Users can ask AI assistants like Claude questions such as "What research projects are on Molecule?" and get answers from live data.

#### What is MCP?

Model Context Protocol is an open standard that allows AI assistants to utilize external tools, enabling access to current data from sources like Molecule's datasets.

### MCP Server Functionality

The MCP server bridges AI assistants and Molecule Protocol data. It lets AI fetch real data about research projects and their activity, rather than relying on pre-trained knowledge.

#### Example Interaction

* **Query**: "What's the latest activity on longevity research projects?"
* **Process**:
  1. AI Assistant interprets the request.
  2. Selects the appropriate tool.
  3. Calls the Molecule MCP server at `https://molecule-mcp.vercel.app/mcp`.

#### Data Sources

The MCP server pulls its data from the Molecule API (GraphQL).

### Install

The server is a remote [Streamable HTTP](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports#streamable-http) endpoint with no authentication, so every client only needs the URL:

```
https://molecule-mcp.vercel.app/mcp
```

Pick your client below. Once connected, your client lists the tools the server currently exposes — ask something like _"What research projects are on Molecule?"_ to check they are loaded.

{% hint style="info" %}
Looking to **create Labs and upload data** from an agent rather than read ecosystem data? That is the [Molecule Skill](molecule-skill.md), a separate local MCP server that holds your credentials and signs transactions.
{% endhint %}

{% tabs %}
{% tab title="Claude (web & desktop)" %}
Remote servers are added as a **custom connector**, not in `claude_desktop_config.json` (that file is for local stdio servers only).

1. Open **Customize → Connectors** and click **+ → Add custom connector**.
2. Name it `Molecule` and paste `https://molecule-mcp.vercel.app/mcp` as the URL. Leave the OAuth settings empty.
3. Click **Add**, then enable the connector in a chat from the **+** menu.

Available on Free (one custom connector), Pro, Max, Team and Enterprise. On Team and Enterprise, an Owner adds it for the organization under **Organization settings → Connectors**. The connector is reached from Anthropic's cloud and works in both claude.ai and Claude Desktop.
{% endtab %}

{% tab title="Claude Code" %}
```bash
claude mcp add --transport http molecule https://molecule-mcp.vercel.app/mcp
```

Add `--scope user` to make it available in every project, or `--scope project` to write it to a shared `.mcp.json` at the repo root:

```json
{
  "mcpServers": {
    "molecule": {
      "type": "http",
      "url": "https://molecule-mcp.vercel.app/mcp"
    }
  }
}
```

Run `/mcp` inside Claude Code to check it's connected.
{% endtab %}

{% tab title="ChatGPT" %}
ChatGPT connects to remote MCP servers through **developer mode** (Plus, Pro, Business, Enterprise and Education, on the web).

1. Turn on developer mode in **Settings → Security and login → Developer mode**.
2. Go to [chatgpt.com/plugins](https://chatgpt.com/plugins), click **+** and create an app.
3. Name it `Molecule`, paste `https://molecule-mcp.vercel.app/mcp` as the MCP server URL and choose **No authentication**.
4. The app appears under **Drafts** — select it in a chat to use the tools.

On Business and Enterprise workspaces, an admin may need to allow developer mode first. See OpenAI's [developer mode guide](https://developers.openai.com/api/docs/guides/developer-mode).
{% endtab %}

{% tab title="Cursor" %}
Add to `~/.cursor/mcp.json` (all projects) or `.cursor/mcp.json` (this project):

```json
{
  "mcpServers": {
    "molecule": {
      "url": "https://molecule-mcp.vercel.app/mcp"
    }
  }
}
```

Then check **Cursor Settings → MCP** shows the server with its tools.
{% endtab %}

{% tab title="VS Code" %}
For GitHub Copilot agent mode, add to `.vscode/mcp.json` in your workspace (note the top-level key is `servers`):

```json
{
  "servers": {
    "molecule": {
      "type": "http",
      "url": "https://molecule-mcp.vercel.app/mcp"
    }
  }
}
```

To add it for every workspace instead, run **MCP: Open User Configuration** from the Command Palette and add the same entry.
{% endtab %}

{% tab title="Codex" %}
```bash
codex mcp add molecule --url https://molecule-mcp.vercel.app/mcp
```

or add it to `~/.codex/config.toml` yourself:

```toml
[mcp_servers.molecule]
url = "https://molecule-mcp.vercel.app/mcp"
```
{% endtab %}

{% tab title="Gemini CLI" %}
```bash
gemini mcp add --transport http molecule https://molecule-mcp.vercel.app/mcp
```

This writes to the project's `.gemini/settings.json` by default; add `-s user` for `~/.gemini/settings.json`. To edit the file yourself, use `httpUrl` — plain `url` means the older SSE transport:

```json
{
  "mcpServers": {
    "molecule": {
      "httpUrl": "https://molecule-mcp.vercel.app/mcp"
    }
  }
}
```
{% endtab %}

{% tab title="Windsurf" %}
Windsurf (now Devin Desktop) has no one-click install. In the Cascade panel, open the **…** menu → **Open MCP config file** and add (note the key is `serverUrl`):

```json
{
  "mcpServers": {
    "molecule": {
      "serverUrl": "https://molecule-mcp.vercel.app/mcp"
    }
  }
}
```

Refresh the MCP list in Cascade after saving.
{% endtab %}

{% tab title="Code (any MCP client)" %}
Connect with the Streamable HTTP transport:

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

See [Programmatic Integration](#programmatic-integration) for a full example with a model.
{% endtab %}
{% endtabs %}

### Example Conversations

* **Ecosystem Overview**: "What's happening in the Molecule ecosystem?"
* **Project Deep Dive**: "Summarize a longevity research project on Molecule and what it's working on."
* **Recent Activity**: "Which research projects have posted updates this month?"

### Self-Hosting

For private deployments or custom configurations:

#### Requirements

* Node.js 18 or higher
* Vercel account or any Node.js platform
* Molecule API key

#### Deploy Steps

1. Request access to the MCP server source from the Molecule team (the repository is not publicly listed).
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

### Caching

The server caches upstream responses in Redis for a few minutes, enhancing performance and abiding by upstream limits — expect data freshness in the minutes range rather than real-time.

### Rate Limits

The public endpoint is rate-limited. Deploy your own instance for heavy or latency-sensitive workloads.

### Programmatic Integration

For AI applications requiring Molecule data, connect to the MCP endpoint directly. Any MCP-compatible client works — the example below uses the Vercel AI SDK's MCP client.

```javascript
import { experimental_createMCPClient as createMCPClient, generateText } from 'ai';
import { openai } from '@ai-sdk/openai';
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js';

const mcpClient = await createMCPClient({
  transport: new StreamableHTTPClientTransport(
    new URL('https://molecule-mcp.vercel.app/mcp')
  )
});
const tools = await mcpClient.tools();

const result = await generateText({
  model: openai('gpt-4o'),
  tools,
  prompt: 'Which longevity research projects have posted updates this month?'
});
```

### Troubleshooting

* Ensure configuration file syntax is valid.
* First request may be slower; consider using Redis for caching.
* Deploy your own instance if facing rate limit errors.

### Resources

* **Public Endpoint**: [https://molecule-mcp.vercel.app/mcp](https://molecule-mcp.vercel.app/mcp)
* **MCP Specification**: [https://modelcontextprotocol.io](https://modelcontextprotocol.io)
* **Source Code & API Key Request**: Contact the Molecule team.

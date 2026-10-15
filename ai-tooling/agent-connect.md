---
description: >-
  Connect Claude, ChatGPT, Codex, Grok or Cursor to a Lab you own: paste one
  address into the app and approve the connection in your browser.
icon: plug
---

# Connect your agent

Agent Connect lets you work in your Lab from Claude, ChatGPT, Codex, Grok or Cursor: you paste one address into the app and approve the connection in your browser, with no keys and no scripts. The app gets its own agent, with its own wallet and a role on each Lab you connect it to.

If you write code against the Labs API, or want an agent that holds its own keys and can create Labs, use [Getting Started](../api-reference/getting-started/README.md) or the [Molecule Skill](molecule-skill.md) instead.

{% hint style="warning" %}
**Not live yet:** Agent Connect is not available yet. The address on this page does not answer, and the Molecule page that approves a connection does not exist yet, so none of the steps below can be completed. Every other part of this page that is not live is marked like this.
{% endhint %}

## The address

Paste this address into your app exactly as it is:

```
https://mcp.molecule.xyz
```

Nothing goes after it: no `/mcp` and no trailing slash. Some apps' own instructions add `/mcp` to a server address; leave it out for Molecule.

{% hint style="info" %}
**Testing on Base Sepolia?** Use `https://staging.mcp.molecule.xyz` instead. It connects your app to Labs in the test app, [testnet.app.molecule.xyz](https://testnet.app.molecule.xyz), which runs on Base Sepolia with test funds only.
{% endhint %}

{% hint style="warning" %}
**Not live yet:** neither address answers yet.
{% endhint %}

## Add Molecule to your app

Pick your app below. Apps change these screens often: if yours looks different, follow the link in its section to the app's own instructions.

{% hint style="warning" %}
**Not live yet:** no app can finish these steps until the address answers.

Each set of steps ends when your app opens a browser window on the Molecule app. Carry on there with [Approve the connection](#approve-the-connection).
{% endhint %}

### claude.ai

Custom connectors work on every Claude plan; the Free plan allows one. Anthropic's guide: [Get started with custom connectors using remote MCP](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp).

On a Free, Pro or Max plan:

1. Open **Customize → Connectors**, click **+ Add**, then **Add custom connector**.
2. Enter a name, such as `Molecule`, and the address, then click **Continue**.
3. Review the authentication settings and click **Continue**. Under **Authentication**, choose **Sign in now**. Under **OAuth client**, keep **Use Claude's published identity (Recommended)**.
4. Click **Add**.

On a Team or Enterprise plan, an owner adds the connector once for everyone, under **Organization settings → Connectors → Add → Custom → Web**, with the same name, address and options. Each member then opens **Customize → Connectors**, finds Molecule (labelled **Custom**) and clicks **Connect**.

To use Molecule in a chat, click **+** at the lower left of the chat, choose **Connectors** and turn Molecule on.

### Claude Code

Add the server for every project (`--scope user`), then sign in from Claude Code. Anthropic's guide: [Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp).

```bash
claude mcp add --transport http --scope user molecule https://mcp.molecule.xyz
```

In Claude Code, run `/mcp`, choose `molecule` and follow the steps in your browser. From your shell, `claude mcp login molecule` does the same.

A connector you added on claude.ai also appears in Claude Code when you sign in to Claude Code with your claude.ai account, and it works as your Claude agent. `claude mcp add` creates a separate Claude Code agent, with its own wallet, its own role on each Lab and its own place in the Lab's limit of 20 agents. Add Molecule in one place unless you want two agents.

### ChatGPT

OpenAI's guide: [Add custom MCP server](https://developers.openai.com/api/docs/guides/custom-mcp-server).

1. In ChatGPT on the web, open **Plugins**, select **+**, then **Add custom MCP server**.
2. Enter a name, such as `Molecule`, and under **Connection** enter the address as the **Server URL**. OpenAI's own example adds a `/mcp` path; leave it out.
3. Choose **OAuth** for authentication, review the risk warning and select **I understand and want to continue**.
4. Select **Create as a plugin**, then install the plugin.

In a chat, type `@` and pick Molecule. If your workspace does not let you add a custom server, ask its admin.

{% hint style="warning" %}
**Not live yet:** whether ChatGPT opens the browser window when you install the plugin or the first time you use it is not confirmed yet.
{% endhint %}

### Codex

Add the server, then sign in. OpenAI's guide: [Model Context Protocol in Codex](https://learn.chatgpt.com/docs/extend/mcp?surface=cli).

```bash
codex mcp add molecule --url https://mcp.molecule.xyz
codex mcp login molecule
```

The Codex CLI, the IDE extension and the ChatGPT desktop app share this setting. To Molecule, Codex and ChatGPT are different apps, so each gets its own agent.

### grok.com

xAI's guide: [Connectors](https://docs.x.ai/grok/connectors).

1. Open **grok.com/connectors** and click **New Connector**, then **Custom**.
2. Enter the address and complete the sign-in.

On a Business or Enterprise team, an admin may have to add the connector first.

### Cursor

Add the server to `~/.cursor/mcp.json` (every project) or `.cursor/mcp.json` (this project). Cursor's guide: [Model Context Protocol](https://cursor.com/docs/mcp).

```json
{
  "mcpServers": {
    "molecule": {
      "url": "https://mcp.molecule.xyz"
    }
  }
}
```

Then open Cursor's **Customize** page, where `molecule` is listed, and sign in when Cursor asks.

{% hint style="warning" %}
**Not live yet:** Cursor has not been tested against Molecule. Depending on how Cursor's desktop app signs in, the Molecule page may show it as an unverified app rather than as Cursor.
{% endhint %}

## Approve the connection

{% hint style="warning" %}
**Not live yet:** the Molecule page described here is not built. Its wording, including the terms sentence, is not final.

When your app opens the browser window:

1. **Sign in to Molecule** with the account you use in the Molecule app. You only do this once in that browser. If you are new to Molecule, you set up your account first and come back to this page.
2. **Choose the Lab.** The page lists the Labs you own that have a data room; if you own none yet, it offers to create one. If the app is already connected to other Labs, the page lists those too: the new Lab is added to the same agent, which then reaches all of them.
3. **Choose a role**, Viewer or Contributor. See [What each role can do](#what-each-role-can-do).
4. **Click Connect.** If you have not accepted Molecule's API Terms of Service and Acceptable Use Policy before, the page asks you to accept them first. Molecule adds the agent to the Lab and pays the network fee. You are asked to confirm only when the Lab is held by an external wallet, which then pays the fee itself, or when your Molecule wallet is not set up to sign without asking.
5. **Go back to your app.** The page tells you when you can close the tab, and the app shows Molecule as connected.

Click **Deny** instead to refuse: the app is told that no access was given.
{% endhint %}

The link your app opens is valid for 15 minutes. After that, start again from your app.

## What each role can do

{% hint style="warning" %}
**Not live yet:** the tools these roles unlock are not built.

* **Viewer** lists and reads the Lab's files, including private files that only members can open.
* **Contributor** does the same and also uploads files, public or private.

To change the role later, remove the agent in Members and connect again with the new role. It comes back as the same agent.
{% endhint %}

Whatever the role, the name, description, tags and searchable text of a private file are public: only its contents are encrypted. And anyone can download a public file as soon as it is uploaded, so treat a public upload as published. Deleting it later does not take back copies already made.

## Tools

{% hint style="warning" %}
**Not live yet:** none of these tools is available yet, and the list is not final.

You don't call tools yourself. Ask in plain words, such as "List the files in my Lab", and the app picks the tool.

* `agent_connect_whoami`: which app and agent you are connected as, and the Labs the agent can reach.
* `lab_lookup`: finds a Lab and shows its public details.
* `data_room_list` (Viewer): lists the files in the Lab's data room.
* `data_room_read` (Viewer): reads one file, including a private one.
* `data_room_upload` (Contributor): uploads a file, public or private.
* `data_room_upload_finish` (Contributor): finishes a large public upload started from Claude Code or Codex.
{% endhint %}

## Upload size

{% hint style="warning" %}
**Not live yet:** the upload limits and the handling of attached files are not final.

In a chat app, the assistant writes the file into its request to Molecule, so an upload has to fit in one of its replies. That suits text, CSV, Markdown and small JSON files of up to a few hundred kilobytes. The server accepts up to 3 MB of file in one upload, but a chat app cannot get near that.

* **A larger public file:** use Claude Code or Codex, which send the file to the data room directly.
* **A larger private file:** upload it in the Molecule app.
* **A file attached to a ChatGPT conversation:** ChatGPT passes it to Molecule as a file, so it does not have to fit in a reply. Files attached on claude.ai or grok.com cannot be uploaded this way.
{% endhint %}

## Disconnecting

Disconnecting is removing the agent from the Lab's members in the Molecule app:

1. Open the Lab's **Members** page, `https://app.molecule.xyz/labs/<shortname>/members`.
2. Find the agent's row. Agents carry an **Agent** tag.
3. Open the row's menu, choose **Remove member**, then **Revoke access**.

Removing the connector in your app does not remove the agent's role on the Lab; only the Members page does. Connecting the same app again later brings back the same agent, with the same wallet.

{% hint style="warning" %}
**Not live yet:** agent rows are not named yet, and removing an agent does not yet end its access.

The agent's row is named after its app, such as **Claude agent** or **Claude Code agent**. An app that chose its own name shows as **Unverified agent:** followed by that name.

The agent loses the Lab as soon as Molecule records the revoke. If it is connected to other Labs, it keeps them and the app stays signed in. Once its last Lab is removed, the app has to sign in again.
{% endhint %}

## Troubleshooting

**The app says Molecule needs authentication.** Connect again from the app. In Claude Code, run `/mcp`, choose `molecule` and select **Re-authenticate**.

**More than 15 minutes passed before you approved.** The link your app opens is valid for 15 minutes. Start again from your app.

**Your Lab already has 20 agents.** That is the most a Lab can have. Remove one in Members, then connect again.

**The connection stopped working after a long break.** A connection that goes unused for 90 days ends, and every connection ends after 365 days. Connect again from your app: the same agent comes back.

{% hint style="warning" %}
**Not live yet:** the two cases below depend on tools and checks that are not built.

**An upload is refused because of the agent's role.** The agent is a Viewer. Remove it in Members, then connect again and choose Contributor.

**The agent says its access was removed after the Lab changed owner.** An agent keeps a Lab only while the person who connected it still owns it. The new owner can connect their own app.
{% endhint %}

## Related

* [Which Tool Do I Need?](README.md): Agent Connect, MIRA and the Molecule Skill side by side
* [Molecule Skill](molecule-skill.md): the plugin for agents that hold their own keys and create Labs
* [Roles & Permissions](../technical-deep-dive/roles-and-permissions.md): what Viewer and Contributor mean on a Lab
* [Data Privacy & Access](../technical-deep-dive/data/data-privacy-and-access.md): how private files stay private
* [Agent access](../api-reference/getting-started/agent-as-a-lab-contributor.md): the developer path, where you give an agent a role yourself

{% include "../.gitbook/includes/support.md" %}

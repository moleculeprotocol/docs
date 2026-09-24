---
description: Pick the right AI tool for what you want to do with Molecule
icon: signs-post
---

# Which Tool Do I Need?

Molecule offers three AI tools. They don't overlap — pick by what you want to do:

| I want to…                                                                        | Use                                         | Runs                   | Setup                                                    |
| --------------------------------------------------------------------------------- | ------------------------------------------- | ---------------------- | -------------------------------------------------------- |
| Chat about Molecule, DeSci and research projects in a ready-made assistant        | [MIRA](mira.md)                             | Web app                | None                                                     |
| Ask my own AI assistant (Claude, ChatGPT, Cursor…) about the Molecule ecosystem   | [Ecosystem Data MCP](ecosystem-data-mcp.md) | Hosted by Molecule     | Paste one URL — no credentials, read-only, free          |
| Have an AI agent create a Lab, upload research files or manage roles              | [Molecule Skill](molecule-skill.md)         | Locally on your machine | Wallet, `mol_` credential and funds for x402 payments and gas |

**Rule of thumb:** if you only need to _read_ ecosystem data, use the Ecosystem Data MCP. If the agent needs to _write_ — create or change a Lab — use the Molecule Skill.

{% hint style="info" %}
The Molecule Skill also contains an MCP server, but it is a separate, local server that holds your credentials and signs transactions. It is not the same as the hosted Ecosystem Data MCP, and you can install both side by side.
{% endhint %}

# Base Note ERP plugin

Connect ChatGPT, Codex or Claude to Base Note ERP with your own ERP account. Your ERP role controls which records and files you can access and which changes you can make. The package includes the staging connector and 16 workflow skills.

## ChatGPT

Open Plugins, select **Add → Add custom MCP server**, and enter `https://staging-alchemy-api.basenotesolutions.com/mcp`. Choose OAuth with dynamic client registration and sign in to the ERP. In a chat, select Base Note ERP. Personal ChatGPT Pro supports reads. Eligible managed workspaces can also enable writes and file uploads.

## Codex

Install the connector and skills from the marketplace.

```sh
codex plugin marketplace add Basenote-Solutions/erp-claude-plugin
codex plugin add base-note-erp@base-note
```

Open a new Codex chat and authenticate the Base Note ERP server when prompted. For direct MCP setup in Codex desktop or the CLI, see [AI connections in the ERP](https://alchemy.basenotesolutions.com/preferences/ai-connections).

## Claude

In claude.ai or Claude Desktop, open Customize, then Plugins, choose Add, Add marketplace, then Add
from a repository, and enter `Basenote-Solutions/erp-claude-plugin`. Press Sync, then Add beside
Base Note ERP.

To take a newer version later: Add, Manage marketplaces, the menu beside erp-claude-plugin, Check
for updates.

In Claude Code:

```bash
claude plugin install base-note-erp --marketplace Basenote-Solutions/erp-claude-plugin
```

The ERP publication workflow maintains the plugin files and the OpenAI marketplace catalog. All rights reserved.

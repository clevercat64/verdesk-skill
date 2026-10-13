# verdesk-skill

The manual that teaches an AI agent to operate a Windows PC and its browsers through [Verdesk](https://verdesk.app), a local MCP server that sends what changed on screen instead of a full screenshot every turn. Works with Claude Code and other MCP clients.

## Use it

1. Install Verdesk from [verdesk.app](https://verdesk.app) (the setup `.exe` is on [GitHub Releases](https://github.com/clevercat64/verdesk/releases/latest)) and run it.
2. In Verdesk, open **Settings → Connection** and press **Copy prompt**, then paste it into your agent. Or register the server yourself:

   ```bash
   claude mcp add --transport http --scope user verdesk http://127.0.0.1:47802/mcp
   ```

3. Reopen your agent's terminal so it loads the MCP server.

You do not need to clone this repository. Verdesk serves this manual from the running program (the `verdesk_skill` tool) and refreshes its own copy in `%USERPROFILE%\.claude\skills\verdesk\` every time it starts, so the manual always matches the installed version.

## What is here

- [`SKILL.md`](./SKILL.md) — the router. If the tools are missing, start with [Get Verdesk](./SKILL.md#get-verdesk-if-the-tools-arent-there); for how the copy is kept current, see [Self-integration & setup](./SKILL.md#self-integration--setup).
- [`references/`](./references) — one file per area: `browser.md`, `desktop.md`, `trains.md` and `verbs.md`. The router says which one to read.
- `plugin.json` — the manifest for Claude Code.

## License

This skill is MIT-licensed ([LICENSE](./LICENSE)). Verdesk itself is covered by its own [EULA](https://github.com/clevercat64/verdesk/blob/main/LICENSE).

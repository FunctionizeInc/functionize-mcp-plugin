# functionize-mcp-plugin

A Claude Code plugin that bundles the hosted Functionize MCP connection
together with skills for driving it well.

Installing this plugin gives you:

- The `functionize-hosted` MCP server connection (all 10 hosted tools:
  session control, streaming, team listing, file attachments).
- Skills that teach Claude how to use those tools correctly: reading a
  session's real status, verifying a run actually passed, sequencing
  gotchas across the tools, and whatever else the team packages in here.

This repo is the distribution package; the server itself is implemented
and operated separately by Functionize.

## Install

```
/plugin marketplace add FunctionizeInc/functionize-mcp-plugin
/plugin install functionize-mcp@functionize-mcp-plugin
```

Then sign in once via `/mcp` (Functionize account, browser OAuth, no
copy/paste).

This works from the terminal CLI, and equally from the **Code** tab of the
Claude Desktop app (v1.2581.0+): click **+** next to the prompt box, choose
**Plugins → Add plugin**, and search the same marketplace. No terminal
needed there, just the GUI plugin browser.

## Claude Desktop — Chat tab

The Code tab above covers Desktop for anyone doing agent-orchestration work
from a coding session. The **Chat** tab is a separate surface with its own
connection system (Connectors, not plugins), so it needs a different setup:
a small local bridge (`mcp-remote`) that talks to the same hosted server.

**Prerequisite:** Node.js 18+ (`node --version`).

1. Open `~/Library/Application Support/Claude/claude_desktop_config.json`
   (create it if it doesn't exist) and add this inside `mcpServers`, keeping
   any entries already there:

   ```json
   {
     "mcpServers": {
       "functionize-hosted": {
         "command": "npx",
         "args": ["-y", "mcp-remote", "https://mcp.functionize.com/mcp"]
       }
     }
   }
   ```

   If a bare `"npx"` doesn't launch (Desktop starts commands without your
   shell, so an nvm/fnm/volta `npx` *shell function* won't resolve), replace
   it with the absolute path from `which npx` or `command -v npx`.

2. Fully quit Claude Desktop (Cmd+Q, not just closing the window) and reopen it.
3. A browser tab opens to sign in with your Functionize account. Complete it.
4. Smoke test: ask Claude *"list my Functionize agent sessions."*

**Troubleshooting:**

| Symptom | Fix |
|---|---|
| Server not listed after restart, or "command not found" | The `command` path isn't executable — use the absolute `npx` path, not a bare `"npx"`. |
| No browser tab opens | Run the bridge by hand to watch the handshake: `npx -y mcp-remote https://mcp.functionize.com/mcp` |
| Need to re-login or clear a stale token | `mcp-remote` caches tokens in `~/.mcp-auth` — remove that directory and restart Desktop. |

## Contributing a skill

See [`skills/README.md`](skills/README.md).

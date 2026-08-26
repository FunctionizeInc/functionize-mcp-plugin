# functionize-mcp-plugin

A Claude Code plugin that bundles the hosted Functionize MCP connection
together with skills for driving it well.

Installing this plugin gives you:

- The `functionize-hosted` MCP server connection (all 10 hosted tools:
  session control, streaming, team listing, file attachments).
- Skills that teach Claude how to use those tools correctly: reading a
  session's real status, verifying a run actually passed, sequencing
  gotchas across the tools, and whatever else the team packages in here.

The server itself lives in
[`functionize-mcp-go`](https://github.com/FunctionizeTeam/functionize-mcp-go).
This repo is the distribution package, not the server implementation.

## Install

```
/plugin marketplace add FunctionizeTeam/functionize-mcp-plugin
/plugin install functionize-mcp@functionize-mcp-plugin
```

Then sign in once via `/mcp` (Functionize account, browser OAuth, no
copy/paste) — see
[`functionize-mcp-go`'s connect doc](https://github.com/FunctionizeTeam/functionize-mcp-go/blob/main/docs/CONNECT-HOSTED-MCP.md)
for the full walkthrough if anything's unclear.

## Contributing a skill

See [`skills/README.md`](skills/README.md).

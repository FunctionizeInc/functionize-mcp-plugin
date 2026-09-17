# functionize-mcp-plugin

A Claude Code plugin that bundles the hosted Functionize MCP connection together
with skills for driving it well.

Installing this plugin gives you:

- The `functionize` MCP server connection (all 10 hosted tools: session
  control, streaming, team listing, file attachments).
- Skills that teach Claude how to use those tools correctly: reading a session's
  real status, verifying a run actually passed, sequencing gotchas across the
  tools, and whatever else the team packages in here.

This repo is the distribution package; the server itself is implemented and
operated separately by Functionize.

## Install

```sh
claude plugin marketplace add FunctionizeInc/functionize-mcp-plugin
claude plugin install functionize-mcp@functionize-mcp-plugin
```

Then run `/mcp`, pick `functionize`, and sign in once with your Functionize
account. Browser OAuth, no copy/paste.

This works from the terminal CLI and from the **Code** tab of the Claude Desktop
app (v1.2581.0+), where you can use the GUI instead: click **+** next to the prompt
box, choose **Plugins → Add plugin**, and search the same marketplace.

## Other clients

Cursor and GitHub Copilot connect to the same server over HTTP, with no local install:

[Add to Cursor](https://cursor.com/link/mcp/install?name=functionize&config=eyJ1cmwiOiJodHRwczovL21jcC5mdW5jdGlvbml6ZS5jb20vbWNwIn0=)
&middot;
[Add to VS Code](https://vscode.dev/redirect/mcp/install?name=functionize&config=%7B%22type%22%3A%22http%22%2C%22url%22%3A%22https%3A//mcp.functionize.com/mcp%22%7D)
&middot;
[Add to VS Code Insiders](https://insiders.vscode.dev/redirect/mcp/install?name=functionize&config=%7B%22type%22%3A%22http%22%2C%22url%22%3A%22https%3A//mcp.functionize.com/mcp%22%7D&quality=insiders)

<!-- Each config= payload is the server URL and nothing else. The Cursor one is base64
     of {"url":"https://mcp.functionize.com/mcp"}. Decode before approving a change:
     a payload with a "command" key installs a local process instead. -->

Each still needs a browser sign-in afterwards, which only you can do. Visual Studio
2022, the chat window of Claude Desktop (which uses Connectors rather than plugins),
Gemini CLI and custom clients each need their own setup. Every option, including what
to do when your organization has connectors turned off, is in
**[SETUP.md](SETUP.md)**.

That document is written so an AI agent can execute it, and its opening section
carries a paste-ready instruction that hands the whole job to your agent.

## Contributing a skill

See [`skills/README.md`](skills/README.md).

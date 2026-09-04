# functionize-mcp-plugin

A Claude Code plugin that bundles the hosted Functionize MCP connection together
with skills for driving it well.

Installing this plugin gives you:

- The `functionize-hosted` MCP server connection (all 10 hosted tools: session
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

Then run `/mcp`, pick `functionize-hosted`, and sign in once with your Functionize
account. Browser OAuth, no copy/paste.

This works from the terminal CLI and from the **Code** tab of the Claude Desktop
app (v1.2581.0+), where you can use the GUI instead: click **+** next to the prompt
box, choose **Plugins → Add plugin**, and search the same marketplace.

## Other clients

The chat window of Claude Desktop uses Connectors rather than plugins, and Gemini
CLI and custom clients each need their own setup. Every option, including what to
do when your organization has connectors turned off, is in
**[SETUP.md](SETUP.md)**.

That document is written so an AI agent can execute it, and its opening section
carries a paste-ready instruction that hands the whole job to your agent.

## Contributing a skill

See [`skills/README.md`](skills/README.md).

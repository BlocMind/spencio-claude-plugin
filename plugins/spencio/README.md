# Spencio for Claude Code

Use your Spencio expense tracker from Claude Code. Ask about your spending, budgets and subscriptions, compare months, and add, edit or recategorize transactions in plain language.

Your data stays on your phone. This plugin connects Claude Code to Spencio's MCP server (`https://spencio.blocmind.com/mcp`), which forwards each request to the Spencio app on your phone while MCP mode is open, and sends back only the answer. Big changes (5 or more transactions) wait for your OK on the phone, and every change can be undone there.

## Setup

1. On your phone: Spencio › Settings › AI assistants › Start MCP mode, and accept the consent.
2. Install the plugin (see below), start Claude Code, run `/mcp`, choose `plugin:spencio:spencio` and sign in. Type the 6-digit code your phone shows and tap Allow.
3. Ask away: "What did I spend on food this month?"

Keep the Spencio app open while you chat. If Claude says Spencio isn't reachable, open the app and keep MCP mode on.

## Install

From the Spencio marketplace on GitHub:

```bash
claude plugin marketplace add BlocMind/spencio-claude-plugin
claude plugin install spencio@spencio-plugins
```

Or inside Claude Code: `/plugin marketplace add BlocMind/spencio-claude-plugin`, then `/plugin install spencio@spencio-plugins`.

## Privacy and support

Spencio's server only relays requests between Claude Code and your phone; it doesn't store what they contain. See the [privacy policy for AI assistants](https://spencio.blocmind.tech/privacy/ai-assistants) and the [setup guide](https://spencio.blocmind.tech/ai-assistants). Questions: support@blocmind.com.

Needs Spencio Plus or the free web dashboard trial.

+++
date = "2026-04-27T09:00:00+12:00"
title = "My Agent's Shell Commands, On the Record"
categories = ["AI", "agents"]
tags = ["AI agents", "agent-receipts", "OpenClaw", "cryptography", "audit trail", "open source"]
draft = false
author = "Otto Jongerius"
+++

A few weeks back I [showed cryptographic receipts for AI agent actions](/post/auditing-github-mcp-agent-receipts/) — through an MCP signing proxy, watching every call to the GitHub MCP server. The proxy works for what flows through MCP. Plenty doesn't.

[OpenClaw](https://github.com/openclaw) is the dangerous place: it's where agents execute shell commands, read and write files, hit APIs. The blast radius of an AI agent is bounded by the tools it can call, and OpenClaw's the runtime that hands them out. If you're going to have an audit trail anywhere, it's there.

The good news: instrumentation can be complete. [@agnt-rcpt/openclaw](https://github.com/agent-receipts/openclaw) hooks into OpenClaw's tool-call lifecycle, which fires for every call regardless of source — MCP or built-in or custom plugin. There's no path the agent can take that doesn't pass through the hooks. No sidestepping.

I tried it on Max Shannon, my Telegram-based agent running on EC2. One session: seven shell commands, thirteen signed receipts. Mid-session the agent verified the plugin worked by querying its own audit trail — and that query has its own receipt.

Here's what showed up on disk.

---

## The setup

[Agent Receipts](https://agentreceipts.ai) is an open protocol that turns every AI agent action into a W3C Verifiable Credential. The [OpenClaw plugin](https://github.com/agent-receipts/openclaw) (`@agnt-rcpt/openclaw`) hooks into [OpenClaw](https://github.com/openclaw), intercepts every tool call, classifies it against a taxonomy, signs a receipt, and stores it in a local SQLite database.

My setup: an EC2 instance running OpenClaw as a systemd service (`openclaw-gateway.service`), with Max Shannon as the test subject. Run a session, see what comes out.

---

## Seven high-risk commands, on the record

Here's what I asked Max to do:

![Telegram message: "I'm testing the agent-receipts project, particularly the openclaw plugin I wrote. I just installed it, can you make some tool calls for me to verify it worked?"](/images/post/max-shannon-prompt.png)

Two minutes later, here's what was on disk:

```
Total receipts: 13  |  Chains: 2
Risk: high: 7, low: 4, medium: 2
Status: success: 13

    #  ACTION                          RISK      STATUS    TARGET                TIMESTAMP
-----------------------------------------------------------------------------------------------------
    1  system.command.execute          high      success   exec                  2026-04-26T21:09:27Z
    2  system.command.execute          high      success   process               2026-04-26T21:09:43Z
    3  system.command.execute          high      success   process               2026-04-26T21:09:48Z
    4  system.command.execute          high      success   exec                  2026-04-26T21:09:55Z
    5  system.command.execute          high      success   process               2026-04-26T21:10:06Z
    6  system.command.execute          high      success   exec                  2026-04-26T21:10:09Z
    7  system.command.execute          high      success   process               2026-04-26T21:10:24Z
    8  unknown                         medium    success   ar_query_receipts     2026-04-26T21:10:40Z
    9  filesystem.file.read            low       success   read                  2026-04-26T21:10:46Z
   10  unknown                         medium    success   ar_query_receipts     2026-04-26T21:11:01Z
    1  filesystem.file.read            low       success   read                  2026-04-26T21:41:38Z
    2  filesystem.file.read            low       success   read                  2026-04-26T21:48:20Z
    3  filesystem.file.read            low       success   read                  2026-04-26T22:18:34Z
```

Seven `system.command.execute` calls in the first chain — shell execution, classified high-risk by the plugin's taxonomy. Six more receipts after that, including some lower-risk file reads in a second session.

Every row is a W3C Verifiable Credential on disk: Ed25519-signed, hash-chained to the receipt before it, independently verifiable with the public key. No trust in the agent, no trust in the plugin — just signatures.

---

## The self-verification moment

Look at rows 8 and 10: `ar_query_receipts`.

Max — asked to verify the plugin worked — chose to do it by querying its own audit trail, then reported back:

![Max Shannon's reply: "Plugin Verified!" with the retrieved receipt's ID, action (filesystem.file.read), risk level (low), status (success), and timestamp (2026-04-26T21:10:46.384Z)](/images/post/max-shannon-verification.png)

Row 8 is that query. Row 9 is the receipt Max retrieved (`filesystem.file.read` at `21:10:46Z` — match the timestamp on the screenshot). Row 10 is Max running the query again to confirm. The agent's act of checking its audit trail is itself in the audit trail.

The `unknown` classification on rows 8 and 10 is a minor taxonomy gap ([openclaw#98](https://github.com/agent-receipts/openclaw/issues/98)) — the plugin's own tools aren't mapped in the action taxonomy yet. The receipts exist either way, signed and chained correctly. The self-verification loop closes.

That's exactly what the protocol is built for: agents that can audit themselves, with that audit attempt itself on the record. It worked on the first real session.

---

## Tunable transparency: hashes by default, plaintext when you want it

By default, the plugin stores a SHA-256 hash of the tool call parameters. You get cryptographic proof of what was passed without the plaintext — agents handle secrets all the time, and you don't want them in your audit log.

When you do want the plaintext, set `parameterPreview: "high"` in `openclaw.json`:

```jsonc
{
  "plugins": {
    "entries": {
      "openclaw-agent-receipts": {
        "config": {
          "parameterPreview": "high"
        }
      }
    }
  }
}
```

Now high-risk actions store both the hash and the actual command:

```json
{
  "action": {
    "type": "system.command.execute",
    "risk_level": "high",
    "parameters_hash": "sha256:9c84a8c9e89a07ff323b0ad52972f148b7f2f5240817f2d9f9892ca514b4522c",
    "parameters_preview": {
      "command": "echo \"Testing agent-receipts plugin fix\""
    }
  }
}
```

Hash for integrity, plaintext for forensics, operator-controlled. The dial accepts `false` (default, hash only), `true` (plaintext for everything), `"high"` (plaintext for high and critical risk), or an array of specific action types.

---

## What I take away

What was new this time wasn't the receipts — it was the completeness. Install one plugin, get a verifiable trail of every tool call the agent makes. And the moment that made me grin: the agent reaching for the audit trail itself mid-session.

Seven shell commands. Thirteen signed receipts. Two chains anyone with my public key can verify. And one of those receipts is the agent looking at its own.

That'll do.

---

- Plugin repo: [github.com/agent-receipts/openclaw](https://github.com/agent-receipts/openclaw)
- Protocol spec + SDKs: [github.com/agent-receipts/ar](https://github.com/agent-receipts/ar)
- Docs: [agentreceipts.ai](https://agentreceipts.ai)

+++
date = "2026-04-27T09:00:00+12:00"
title = "My Agent's Shell Commands, On the Record"
categories = ["AI", "agents"]
tags = ["AI agents", "agent-receipts", "OpenClaw", "cryptography", "audit trail", "open source"]
draft = false
author = "Otto Jongerius"
+++

A few weeks ago I [asked](/post/your-ai-agent-just-sent-an-email/) what it would mean to have proof of what an AI agent actually did on your behalf — not vendor logs or a chat scrollback, but a signed, tamper-evident record.

Last weekend I got to see one. Max Shannon — a Telegram-based AI agent I run on EC2 — sat down for a session and executed seven shell commands. Every one came back out as a cryptographically signed receipt, classified high-risk, hash-chained in sequence. Mid-session, the agent asked to see its own audit trail — and that query has its own receipt too.

That's what proof looks like in practice. Here's what showed up on disk.

---

## The setup

[Agent Receipts](https://agentreceipts.ai) is an open protocol that turns every AI agent action into a W3C Verifiable Credential. The [OpenClaw plugin](https://github.com/agent-receipts/openclaw) (`@agnt-rcpt/openclaw`) hooks into [OpenClaw](https://github.com/openclaw), intercepts every tool call, classifies it against a taxonomy, signs a receipt, and stores it in a local SQLite database.

My setup: an EC2 instance running OpenClaw as a systemd service (`openclaw-gateway.service`), and Max Shannon — my Telegram-based agent — as the test subject. Run a session, see what comes out.

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

Seven `system.command.execute` calls in the first chain — shell execution, classified high-risk by the plugin's taxonomy mapper before the receipt was even built. Twelve more receipts after that, including some lower-risk file reads in a second session.

Every row is a W3C Verifiable Credential on disk: Ed25519-signed, hash-chained to the receipt before it, independently verifiable with the public key. No trust in the agent, no trust in the plugin — just signatures.

This is the part I'd been waiting to see. Watching my agent act, and watching the actions show up in a format I can verify.

---

## The self-verification moment

Look at rows 8 and 10: `ar_query_receipts`.

Max — unprompted — called the plugin's own receipt query tool mid-session and reported back:

![Max Shannon's reply: "Plugin Verified!" with the retrieved receipt's ID, action (filesystem.file.read), risk level (low), status (success), and timestamp (2026-04-26T21:10:46.384Z)](/images/post/max-shannon-verification.png)

Row 8 is that query. Row 9 is the receipt Max retrieved (`filesystem.file.read` at `21:10:46Z` — match the timestamp on the screenshot). Row 10 is Max running the query again to confirm. The agent's act of checking its audit trail is itself in the audit trail.

The `unknown` classification on rows 8 and 10 is a minor taxonomy gap ([openclaw#98](https://github.com/agent-receipts/openclaw/issues/98)) — the plugin's own tools aren't mapped in the action taxonomy yet. The receipts exist either way, signed and chained correctly. The self-verification loop closes.

This is the designed use case. Agents that can audit themselves, with that audit attempt itself on the record. It worked on the first real session.

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

I'd seen Agent Receipts work [through the MCP proxy](/post/auditing-github-mcp-agent-receipts/) on my laptop, watching GitHub API reads get signed and chained. What was new this time: a different agent on a different machine, the plugin approach instead of a proxy, high-risk shell commands instead of low-risk API reads — and the moment that made me grin, the agent reaching for the audit trail itself mid-session.

Seven shell commands. Thirteen signed receipts. One chain that anyone with my public key can verify. And one of those receipts is the agent looking at its own.

That'll do.

---

- Plugin repo: [github.com/agent-receipts/openclaw](https://github.com/agent-receipts/openclaw)
- Protocol spec + SDKs: [github.com/agent-receipts/ar](https://github.com/agent-receipts/ar)
- Docs: [agentreceipts.ai](https://agentreceipts.ai)

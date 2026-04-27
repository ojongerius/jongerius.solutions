+++
date = "2026-04-27T09:00:00+12:00"
title = "Eight Bugs Before My First Receipt"
categories = ["AI", "agents"]
tags = ["AI agents", "agent-receipts", "OpenClaw", "cryptography", "audit trail", "open source"]
draft = false
author = "Otto Jongerius"
+++

I spent two days trialing a v0 tool whose entire purpose is to give you a cryptographically signed receipt for every action your AI agent takes. I filed eight issues across two repos before the first receipt printed to my terminal.

That's not a complaint. That's the story.

---

## What I was testing

[Agent Receipts](https://agentreceipts.ai) is an open protocol for auditing AI agent actions — every tool call gets a signed, hash-chained W3C Verifiable Credential. The [OpenClaw plugin](https://github.com/agent-receipts/openclaw) (`@agnt-rcpt/openclaw`) hooks into [OpenClaw](https://github.com/openclaw), intercepts every tool call, classifies it against a taxonomy, and stores a receipt in a local SQLite database.

My setup: an EC2 instance running OpenClaw as a systemd service (`openclaw-gateway.service`), and Max Shannon — a Telegram-based AI agent — as the test subject.

The goal was simple: run Max through a session and see what the receipts look like.

---

## Bug 1: the plugin wouldn't install

The OpenClaw installation page on agentreceipts.ai was "Coming soon" at the time, despite the plugin being live on npm ([ar#247](https://github.com/agent-receipts/ar/issues/247)). So I went straight to the package and ran:

```bash
openclaw plugins install @agnt-rcpt/openclaw
```

Fails. The `extensions` field in the plugin's `package.json` pointed at `./src/index.ts` — the TypeScript source — instead of `./dist/src/index.js`. Every install was broken for everyone, out of the box.

Filed [openclaw#85](https://github.com/agent-receipts/openclaw/issues/85). Merged. Released as 0.3.1.

Reinstall succeeds. Add the plugin's tools to `alsoAllow` in `openclaw.json` (vim, not nano — the `openclaw config edit` command doesn't exist yet, [openclaw#91](https://github.com/agent-receipts/openclaw/issues/91)). Restart the gateway:

```bash
sudo systemctl restart openclaw-gateway
```

(The docs say "restart the gateway" without giving a command. [openclaw#92](https://github.com/agent-receipts/openclaw/issues/92).)

Max runs through a session. The gateway logs confirm receipts are being written to SQLite. Progress.

---

## Then the CLI was silent

```bash
npx @agnt-rcpt/openclaw receipts
```

Nothing. No output. No error. Just a blinking cursor and the quiet humiliation of knowing the receipts are right there in the database.

This took a while to diagnose. Two separate bugs:

**Bug: `params: string[]` — LIMIT passed as TEXT**

The `store.query()` method in the SDK was passing the LIMIT value as a string instead of an integer. On Node v22, SQLite returns zero rows when LIMIT is bound as TEXT. Fixed in [ar#249](https://github.com/agent-receipts/ar/issues/249), merged.

**Bug: `isDirectRun` check doesn't follow symlinks**

The CLI uses `path.resolve()` to check whether it's being invoked directly or required as a module. `path.resolve()` doesn't follow symlinks. When you run the CLI via `npx` — which creates a symlink — the check fails silently, and the CLI does nothing.

This one's still being fixed: [fix/cli-entrypoint-symlink](https://github.com/agent-receipts/openclaw/tree/fix/cli-entrypoint-symlink) branch open, targeting 0.3.3. I confirmed it by running `node cli.js` directly on the EC2 instance — receipts appeared immediately. Patched the dist directory by hand to confirm, then called it done.

(There are also three documentation bugs — [openclaw#90](https://github.com/agent-receipts/openclaw/issues/90), [openclaw#91](https://github.com/agent-receipts/openclaw/issues/91), [openclaw#92](https://github.com/agent-receipts/openclaw/issues/92) — but those don't block anything, they just slow you down.)

---

## What the receipts actually showed

After patching the dist and running `receipts`, here's what came out:

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

Seven high-risk `system.command.execute` calls in the first chain. A risk classifier correctly tagging shell execution as high. Thirteen receipts, all success, all signed and stored.

That's the tool working.

---

## The moment that made it worth it

Rows 8 and 10: `ar_query_receipts`.

Max — unprompted by me — called the plugin's own receipt query tool mid-session. It asked to see its own audit trail. The plugin caught that call, signed a receipt for it, and chained it in sequence. The agent's act of checking its audit trail is itself in the audit trail.

The `unknown` classification is a minor bug: the plugin's own tools (`ar_query_receipts`, `ar_verify_chain`) aren't in the taxonomy yet ([openclaw#98](https://github.com/agent-receipts/openclaw/issues/98)). But the receipt exists, it's signed, it's hash-chained to the receipts that came before it. The self-verification loop closes.

That's the designed use case. It worked on the first real session.

---

## What parameterPreview adds

By default, the plugin stores a SHA-256 hash of the tool call parameters. You get proof that the parameters weren't tampered with, but no plaintext — you can't reconstruct what `exec` actually ran.

Setting `parameterPreview: "high"` in `openclaw.json` changes that for high-risk actions:

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

The receipt then includes both the hash and the plaintext command:

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

Hash still present. Plaintext alongside it. The operator decides how much detail to trade for privacy — `false` (hashes only), `true` (all actions), `"high"` (high and critical risk only), or an array of specific action types.

---

## Honest take

v0 is rough. The install breaks out of the box. The CLI is silent via npx. The documentation is incomplete. If you're not comfortable digging into `node_modules` and filing GitHub issues, you'll bounce off this immediately.

But the design is right.

Hash-chained W3C Verifiable Credentials signed with Ed25519. Privacy-preserving by default — parameters hashed, never stored in plaintext unless you opt in. Framework-agnostic — the plugin hooks into OpenClaw's event system; no MCP required, no assumption about your agent framework. And the self-verification case works as intended, on the first real session.

The yak shave was annoying. The receipts at the end of it were worth seeing.

---

- Plugin repo: [github.com/agent-receipts/openclaw](https://github.com/agent-receipts/openclaw)
- Protocol spec + SDKs: [github.com/agent-receipts/ar](https://github.com/agent-receipts/ar)
- Docs: [agentreceipts.ai](https://agentreceipts.ai)

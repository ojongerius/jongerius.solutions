+++
date = "2026-04-03T09:00:00+12:00"
title = "Your AI Agent Just Sent an Email. Can You Prove It?"
categories = ["AI", "agents"]
tags = ["AI agents", "accountability", "open source", "MCP", "cryptography"]
draft = false
author = "Otto Jongerius"
+++

Last week, I asked an AI agent to clean up some files in a project directory. It did a great job — renamed a few things, deleted some stale configs, updated a README. I know this because I watched it happen in my terminal.

But if you asked me to *prove* what it did? To show you an authoritative record of every action, in order, with cryptographic proof that nothing was altered after the fact? I couldn't.

Nobody can.

---

We're in a strange moment. AI agents are crossing the line from "tools that generate text" to "tools that take actions." They send emails. They modify files. They make API calls. They create pull requests, update databases, manage calendars. Some are starting to make purchases.

Every week, the list of things agents can do on our behalf gets longer. MCP — the Model Context Protocol — is accelerating this by giving agents a standardised way to connect to tools and services. The ecosystem is exploding.

And yet: there is no standard way to record what an agent actually did.

Think about that for a moment. We're handing agents the keys to our digital lives, and the best audit trail we have is... scrolling back through a chat window. Or maybe checking vendor-specific logs that only exist if you opted into a particular observability platform.

---

This isn't a hypothetical problem. It's already real in a few ways:

**You can't reconstruct what happened.** When an agent takes a sequence of actions and something goes wrong — a file deleted that shouldn't have been, an email sent to the wrong person, an API call with bad parameters — you're left piecing together what happened from fragments. Chat logs, terminal output, maybe some application-level logging if you're lucky. There's no single, authoritative record.

**You can't prove what happened.** Even if you *do* have logs, they're not tamper-evident. They're not signed. There's no cryptographic link between one action and the next. If you needed to demonstrate to someone — a colleague, a compliance officer, a regulator — exactly what your agent did and in what order, you have nothing that would hold up to scrutiny.

**You can't compare across agents.** The way Claude Code logs actions is different from how Codex does it, which is different from LangChain, which is different from whatever your team built internally. There's no common format. Every agent is its own silo.

**Regulation is coming, but there's no standard to comply with.** The EU AI Act mandates traceability for high-risk AI systems. Article 12 requires that these systems are designed to allow for automatic recording of events. But the Act doesn't define a record format. It leaves that to implementers. Right now, "compliance" means whatever each vendor decides it means.

---

I keep thinking about an analogy. We've been here before — not with agents, but with media.

A few years ago, the internet had a provenance problem with images and video. Anyone could create, modify, or redistribute media with no way to verify its origin or history. The response was C2PA — the Coalition for Content Provenance and Authenticity — which created Content Credentials: cryptographically signed metadata that travels with a piece of media and records its origin and edit history.

Content Credentials didn't solve every problem. But they established a *primitive* — a signed, verifiable record of what happened to a piece of content. That primitive became something the industry could build on.

We need the same primitive for agent actions.

Not a vendor-specific log. Not an observability dashboard. A cryptographically signed, tamper-evident record of a single action taken by an AI agent on behalf of a human. Something that captures who authorised it, what happened, whether it succeeded, and where it sits in a sequence of actions. Something that any agent framework can produce and any verifier can check.

A receipt.

---

I've been working on this. It's an open protocol called [Agent Receipts](https://agentreceipts.ai) — an open standard for cryptographically signed, tamper-evident records of AI agent actions. The spec is public, there are working SDKs in TypeScript, Python, and Go, and an MCP proxy that can sit in front of any MCP server to generate receipts automatically.

But a protocol is only as good as the community that shapes it. The spec is at v0.1.0 and there are real open questions: how should receipts work when agents fan out into parallel sub-tasks? What does key management look like in practice? How granular should the action taxonomy be?

If this problem resonates with you — whether you're building agents, deploying them in production, or thinking about compliance and governance — I'd love your input:

- **Read the spec** at [Agent Receipts](https://agentreceipts.ai)
- **Open an issue** or join the discussion at [github.com/agent-receipts](https://github.com/agent-receipts)
- **Tell me I'm wrong** — if there's a better approach, I want to hear it

Over the coming weeks I'll be sharing more about the design decisions behind the protocol, and demonstrating it working with tools like Claude Code and Codex. But the spec belongs to the community, not to me.

Because the question stands: as agents become more capable and more autonomous, who's keeping the record?

Right now, nobody is. Let's fix that.

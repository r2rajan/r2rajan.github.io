---
title: "Part-3: Who are you? How client Registration works in the Agentic World"
excerpt: "We've spent years making sure applications register with Oauth. Does this model still work in the Agentic world? What if your agents could carry their own digital key and walk straight past it? A look at DCR and CIMD and when each earns its place in your agent architecture."
categories:
  - Agentic Identity
  - AI
tags:
  - genai
  - AI
  - iam
  - llm
layout: single
toc: true
toc_sticky: true
mermaid: true
sidebar:
  nav: categories
---

# Part-3: Who Are You? How Client Registration works in the Agentic World

Everytime i go to Las Vegas for attending technology conferences, I am always worried about the soul-crushing, velvet-rope maze at the hotel front desk where you stand for forty minutes to register and prove your identity to get that plastic key card. 

Recently, I landed in Las Vegas in December 2025 to speak at AWS's flagship conference **re:invent**. With over 50,000 in-person attendees descending on the city in a single week, this time as I walked into the lobby **and** I didn't stop. While a sea of people stood in line to **register** by handing over IDs, waiting for the front desk to manually type data into a database, and receive a piece of plastic, I kept moving. My room was assigned, my identity was verified, and my phone was already a functioning key. I bypassed the **front desk** entirely and went straight to my room. It was a frictionless experience.

That plastic key card, in the world of OAuth, we call it Dynamic Client Registration (DCR). We've spent years getting applications to stand in that same "front desk" line, registering with an authorization server, waiting for credentials, storing them in a database. In my [previous post](https://r2rajan.github.io/agentic%20identity/ai/2026/04/21/Whosaidyes/), I walked through how Alice grants consent to individual agents using exactly this model. Each agent registers as its own OAuth client via DCR (RFC 7591), getting per-agent revocation, independent scope ceilings, and clean audit trails.

But what if our agents could just walk past the velvet rope? What if they could carry their own **Digital Key** that a server could verify on the fly? No registration desk, no waiting, no database entry.

![Introduction](/assets/images/20260510/Intro.png)

That's the promise of Client ID Metadata Documents (CIMD). This post breaks down both models, when each applies, and how they work in the agentic world.

## The Registration Problem in Agentic Systems

Traditional OAuth assumes a small, known set of clients registered with a small, known set of authorization servers. An admin creates a client in the IdP dashboard, copies the `client_id` and `client_secret`, and hardcodes them into the app config.

Agents break this assumption in two directions:

**Agent-side explosion.** An enterprise might deploy dozens of agents (reporting-agent, coding-agent, deployment-agent), each needing its own identity. Manual registration doesn't scale.

**Server-side explosion.** In MCP, a single agent might connect to hundreds of tool servers. If each server requires separate registration, the agent needs hundreds of `client_id` values, one per server.

DCR and CIMD each address one side of this problem.

## Dynamic Client Registration (DCR): The Server-Managed Model

DCR (RFC 7591) lets a client programmatically register with an authorization server by POSTing its metadata to a `/register` endpoint. The server validates, stores the registration, and returns credentials.

```
POST /register HTTP/1.1
Content-Type: application/json

{
  "client_name": "Reporting Agent",
  "redirect_uris": ["https://agents.example.com/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "scope": "reports:read",
  "agent_metadata": {
    "owner": "data-platform-team",
    "agent_type": "autonomous-reporting",
    "capability_version": "2.3.0"
  }
}
```

The server responds with a unique `client_id` and (for confidential clients) a `client_secret`. From that point forward, the agent authenticates using those credentials.

### Why DCR Works for Agent Fleets

In my [previous post](https://r2rajan.github.io/agentic%20identity/ai/2026/04/21/Whosaidyes/), I made the case for per-agent registration: automated onboarding, scoped revocation, independent scope ceilings, and clean audit attribution. DCR makes all of that practical. Each agent gets its own `client_id`, its own scope ceiling, and its own entry in the registry that doubles as your source of truth for what agents exist and who owns them.

### Where DCR Breaks Down

DCR was designed for a world where the number of clients is bounded and the authorization server is a known entity. In the world of agents and open ecosystems, three problems emerge:

**Unbounded database growth.** If 10,000 users each run the same coding-agent, that's 10,000 registrations for what is logically one application.

**The open endpoint problem.** The `/register` endpoint must be accessible to unauthenticated clients. This makes it a target for DDoS attacks and registration flooding.

**N × M coordination.** If an agent connects to M different MCP servers, each with its own AS, it needs M separate registrations. This is the "registration wall" that blocks agent-to-server connectivity.

To make matters worse, many identity providers (Entra ID, some Okta configurations) don't expose a public DCR endpoint or require a pre-provisioned API key to access the registration_endpoint. Teams end up building OAuth proxy infrastructure just to work around this.

## Client ID Metadata Documents (CIMD): The Client-Hosted Model

CIMD flips the registration model entirely. Instead of the client registering **with** the authorization server (AS), the client hosts its own identity document *for* the AS to fetch.

![cimdflow](/assets/images/20260510/cimdflow.png)

The `client_id` is an HTTPS URL. The authorization server fetches that URL, reads the JSON metadata, and validates it on demand. 

```json
{
  "client_id": "https://coding-agent.example.com/.well-known/oauth-client.json",
  "client_name": "Coding Agent",
  "redirect_uris": [
    "https://coding-agent.example.com/callback",
    "http://localhost:3000/callback"
  ],
  "grant_types": ["authorization_code"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none"
}
```

The agent hosts this file. Any MCP server that supports CIMD can fetch it, validate it, and proceed with the OAuth flow.

### The CIMD Flow

```
1. Agent sends authorization request with client_id = https://coding-agent.example.com/.well-known/oauth-client.json
2. AS fetches that URL via HTTP GET
3. AS validates: client_id in JSON matches the URL, redirect_uris are consistent
4. AS shows consent screen using client_name and logo_uri from the metadata
5. AS caches the metadata (respecting HTTP cache headers)
6. OAuth flow proceeds normally
```

No registration phase and importantly there are no credentials. The AS verifies identity by confirming the agent controls the domain where the metadata lives.

### How CIMD Prevents Impersonation

The obvious question is what stops a malicious agent at `evil.com` from claiming to be `coding-agent.example.com`?

The answer is **redirect_uri validation**. The attacker sends an authorization request using the legitimate agent's `client_id` URL but includes their own `redirect_uri` (`https://evil.com/callback`). The AS fetches the metadata from `coding-agent.example.com`, reads the `redirect_uris` list, and sees that `evil.com` isn't in it. Request denied.

The attacker can't modify the metadata file because they don't control `coding-agent.example.com`. Domain ownership *is* the identity proof.

### Confidential Clients with CIMD

DCR gives confidential clients a `client_secret` from the server. CIMD takes a different approach: the client proves identity using its own private key.

```json
{
  "client_id": "https://coding-agent.example.com/.well-known/oauth-client.json",
  "client_name": "Coding Agent",
  "redirect_uris": ["https://coding-agent.example.com/callback"],
  "token_endpoint_auth_method": "private_key_jwt",
  "jwks_uri": "https://coding-agent.example.com/.well-known/jwks.json"
}
```

When the agent authenticates at the token endpoint, it signs a JWT with its private key. The AS fetches the public key from `jwks_uri` and verifies the signature. No shared secret required.

## When to Use DCR for Agents?

DCR earns its complexity in one specific scenario: you own the authorization server and you need per-instance control over your agent fleet.

If you're running an enterprise where security teams need to revoke a single compromised agent without touching the rest, DCR is your tool. If compliance requires a registry of every authorized agent with its owner, permission history, and lifecycle state, DCR gives you that registry as a byproduct of registration.

The concrete cases:

- You need the AS to enforce scope ceilings at registration time. Reporting-agent caps at `reports:read`. Coding-agent gets `code:read code:write`. The ceiling is set before the user ever sees a consent screen.
- Your agents evolve. Coding-agent ships a new capability next quarter that requires `deployments:trigger`. You want the AS to track that scope escalation and force re-consent. DCR's registration record gives you the audit trail.
- An agent gets compromised at 2am. You revoke its `client_id` and every token tied to it dies immediately. The other 15 agents in your fleet keep running.

The pattern: DCR works when the number of agents is bounded, the AS is a known entity you control, and governance matters more than onboarding speed.

## When CIMD is useful?

For most agent-to-server connections in the MCP world, CIMD should be your default and the reason is that your agent will connect to servers it has never seen before. If each connection requires a registration step, you've built a toll booth on every on-ramp. CIMD removes the toll booth.

Where it shines:

- Your agent connects to 20+ external MCP servers (GitHub tools, Slack tools, monitoring APIs). One metadata file at `https://your-agent.com/oauth.json` works for all of them. No per-server credentials to manage.
- You ship an IDE extension or CLI tool used by 10,000 developers. With DCR, that's 10,000 registrations in every server's database. With CIMD, it's one URL.
- You want new server connections to work the moment a user clicks "connect." No admin tickets, no API keys, no waiting for IT to provision a client.

In short, if your agent needs to talk to servers you don't control, CIMD is the path of least resistance.

## The Hybrid Model: DCR Internally, CIMD Externally

These two models aren't mutually exclusive. They solve different problems at different layers. Now consider this architecture where both DCR and CIMD work together to solve different problems. 

![HybridDCRandCIMD](/assets/images/20260510/hybrid.png)

**Internally:** coding-agent registers via DCR with your enterprise AS. It gets a unique `client_id`, scope ceilings, and all the governance benefits. Alice's consent is managed through the patterns from my previous post,  standing authorization for routine work, task-scoped grants for sensitive operations.

**Externally:** When coding-agent needs to connect to an external MCP server (GitHub tools, Slack tools, third-party APIs), it presents its CIMD. No registration required. The external server fetches the metadata, validates the domain, and proceeds.

The agent holds two identities:
- An internal DCR-issued `client_id` for enterprise governance
- A CIMD URL for open federation

This gives you governance internally and speed externally

## Security Trade-offs

Let's understanad the security threat surface of each model to help you make informed decisions.

### DCR Risks

DCR's biggest exposure is the `/register` endpoint itself. It's open by design, which means an attacker can flood it with junk registrations until your AS database chokes. Rate limiting and requiring initial access tokens help, but you're still defending an open door. Beyond flooding, there's impersonation: nothing stops an attacker from registering with `client_name: "Official Coding Agent"` and tricking users on the consent screen. Software statements can mitigate this, but few teams implement them today. And then there's the long tail problem. Agents get decommissioned, teams move on, but the registrations stay in the database. Without TTLs or periodic cleanup, you accumulate dead entries that bloat storage and complicate audits.

### CIMD Risks

CIMD trades the open registration endpoint for a different risk: your AS now fetches URLs from strangers. A malicious client could submit `https://169.254.169.254/` as its `client_id` and trick your AS into hitting internal infrastructure. You need a hardened fetcher that blocks private IP ranges, enforces timeouts, and caps response size. The localhost problem is subtler: if a client claims `http://localhost:1234` as its identity, the AS can't verify which application is actually listening on that port. In production, restrict CIMD to non-localhost HTTPS URLs. Finally, domain ownership proves identity but not intent. Anyone can register `my-evil-agent.io` and host valid metadata there. The AS knows *who* is asking, but not whether they should be **trusted**. Trust policies, warning messages for unknown domains, and eventually Software Statements are the path forward here.

## What's Next: Software Statements and Platform Attestation

Both DCR and CIMD have a gap: neither proves the agent is *who it claims to be* beyond domain ownership or registration-time trust.

The emerging answer is **Software Statements** (defined in RFC 7591 §2.3). These are signed JWTs issued by a trusted third party (an app store, an OS vendor, a corporate registry) that attest to the agent's identity.

1. coding-agent hosts its CIMD at `https://coding-agent.example.com/oauth.json`
2. The metadata includes a `software_statement`, a JWT signed by your enterprise's agent registry
3. The external MCP server validates the statement against the registry's public key
4. Trust is established not just by domain ownership, but by a verifiable chain of attestation

This bridges the gap between CIMD's scalability and DCR's trust guarantees. The agent gets frictionless connectivity *and* verifiable identity.

## Conclusion

DCR and CIMD aren't competitors. They answer different questions at different trust boundaries. DCR answers "should this agent be allowed to exist in my system?" CIMD answers "how does this agent introduce itself to a server it's never met?"

Use DCR inside your enterprise where you need the AS to gatekeep. Use CIMD at the edges where your agents meet the open world. Most teams will end up running both.

The consent patterns from my [previous post](https://r2rajan.github.io/agentic%20identity/ai/2026/04/21/Whosaidyes/) still apply here. Per-agent scope ceilings, incremental consent, standing vs. task-scoped authorization: all of that operates at the DCR layer, governing what Alice approves and how those approvals evolve over time. CIMD operates one layer below, solving the connectivity problem so your agents can actually reach the servers where those tokens need to work without friction of registration.

---

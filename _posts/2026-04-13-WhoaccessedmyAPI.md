---
title: "Who Called That API? Why AI Agents Need Delegation, Not Impersonation"
excerpt: "When an AI agent acts on behalf of a user, your audit log needs to show both identities, not just one. RFC 8693 token exchange makes that possible with scope-narrowed, audience-restricted delegation tokens."
categories:
  - AgenticIdentity
  - AI
tags:
  - genai
  - iam
  - llm
layout: single
toc: true
toc_sticky: true
mermaid: true
sidebar:
  nav: categories
---
# Who Called That API? Why AI Agents Need Delegation, Not Impersonation

When an AI agent accesses a service on behalf of a user, who shows up in the audit log? If the answer is **just the user** or **just the agent**, you have a gap that will surface during your next security review.

Most agentic platforms today use one of two flawed patterns: the agent impersonates the user (forwarding their token directly), or the agent authenticates as itself with no link to the user who initiated the request. Both patterns break down when you need to answer the question every incident responder asks, **who did what?** When AI agents are acting on behalf of a human user or another agent, you also need to ask another question. **Through which system did they do it?**

This post explains why delegation with dual identity is the correct model for agentic systems, and how OAuth 2.0 Token Exchange (RFC 8693) provides the standard to implement it.

## The Problem with Impersonation

In the impersonation model, the agent receives the user's access token and presents it directly to downstream services. The service sees the user's identity and grants access based on the user's permissions.

![Agent Impersonation Pattern](/assets/images/blog-1-impersonation-pattern.png)

The token that reaches the downstream service looks like this:

```json
{
  "iss": "https://idp.example.com",
  "sub": "alice@example.com",
  "scope": "reports:read tickets:read tickets:write",
  "aud": "https://api.example.com",
  "exp": 1743120000
}
```

This creates three problems.

First, the downstream service cannot distinguish between Alice calling the API directly and an agent calling it on her behalf. The audit trail shows `alice@example.com` for both. During an incident, you cannot determine whether a human or an automated agent performed a specific action.

Second, the agent inherits all of Alice's permissions. If Alice has `tickets:write` scope but the agent only needs `reports:read`, the agent still carries the full scope set. A compromised or misbehaving agent can exercise permissions it was never intended to use.

Third, revoking the agent's access requires revoking Alice's token, which locks Alice out of every system, not just the agent.

## Delegation: The Agent Authenticates as Itself

In the delegation model, the agent does not forward the user's token. Instead, it presents both the user's token and its own identity to a token exchange service. The service issues a new token that carries both identities: who authorized the action (the user) and who performed it (the agent).

![Agent Delegation Pattern](/assets/images/blog-2-delegation-pattern.png)

The delegated token carries dual identity:


```json
{
  "iss": "https://idp.example.com",
  "sub": "alice@example.com",
  "act": {
    "sub": "reporting-agent"
  },
  "aud": "https://api.example.com",
  "scope": "reports:read",
  "exp": 1743120300
}
```

The `sub` claim identifies Alice as the authorizing user. The `act.sub` claim identifies the agent that performed the action. The `aud` claim restricts this token to a specific downstream service. The `scope` is narrowed to only what the agent needs for this particular call.

The downstream service now has everything it needs: authorize based on `sub`, attribute the action to `act.sub`, reject the token if the audience does not match, and log both identities for audit.

## How RFC 8693 Makes This Work

RFC 8693 defines OAuth 2.0 Token Exchange, a standard protocol for exchanging one security token for another. It is the mechanism that turns impersonation into delegation.

![RFC 8693 Token Exchange Flow](/assets/images/blog-3-rfc8693-token-exchange.png)

The exchange takes two inputs:

- A `subject_token`: the user's JWT from the identity provider
- An `actor_token`: the agent's workload identity credential

The token exchange service validates both tokens and issues a new token with narrowed scope. The authorization policy determines how scopes are narrowed. A common pattern for agentic platforms is to compute the intersection of user permissions, agent permissions, and service requirements, ensuring the delegated token never exceeds what any single party allows. The resulting token is scoped to a specific downstream service audience.

Here is what the token exchange request looks like:

```
POST /token HTTP/1.1
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=eyJhbGciOiJSUzI1Alice...   (Alice's JWT)
&subject_token_type=urn:ietf:params:oauth:token-type:jwt
&actor_token=eyJhbGciOiJSUzI1Agent...     (Agent's credential)
&actor_token_type=urn:ietf:params:oauth:token-type:jwt
&audience=https://api.example.com
&scope=reports:read
```

The response is a new token with the dual-identity structure shown above. The token exchange service can enforce several constraints:

1. **Scope narrowing:** the issued token's scope can be limited to only the permissions required for the target service. An agent declared with `reports:read` cannot obtain a token with `tickets:write`, even if the user has that permission.

2. **Audience restriction:** each token is bound to a single downstream service via the `aud` claim. A token minted for the reporting service is rejected by the ticketing service. This limits blast radius if a single service is compromised.

3. **Short TTL**: delegated tokens are issued with short expiration times (typically 5 minutes). The agent caches them and re-requests transparently when they expire. If the agent is compromised, the window of exposure is limited to the token expiry.

## Comparing the Models

| Dimension | Impersonation | Delegation (RFC 8693) |
|---|---|---|
| Audit trail | User identity only | User + agent identity |
| Scope control | Full user permissions | Intersection of user, agent, and service scopes |
| Token audience | Broad (all services) | Per-service restriction |
| Revocation | Revoke user token (locks out user) | Revoke agent identity (user unaffected) |
| Incident response | Cannot distinguish human from agent actions | Full attribution chain |
| Token lifetime | Matches user session (minutes to hours) | Short-lived (5 minutes), auto-refreshed |

## Conclusion

If you are building an agentic platform where AI agents call services on behalf of users, the identity model you choose determines whether your audit trails hold up under scrutiny.

Impersonation is simpler to implement. Forward the user's token and move on. But it creates a blind spot in every audit log, grants agents more permissions than they need, and couples agent lifecycle to user credentials.

Delegation with RFC 8693 requires a token exchange service and per-service audience management. The operational overhead is higher. But it gives you individual attribution on every call, least-privilege enforcement at the token level, and independent revocation of agent and user identities. For security-sensitive environments where compliance, auditability, and least-privilege access are requirements, RFC 8693 token exchange is the foundation to build on.

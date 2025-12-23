---
title: "Understanding OAuth Authentication in Amazon Bedrock AgentCore: A Deep Dive"
excerpt: "A comprehensive deep dive into Amazon Bedrock AgentCore's dual authentication pattern"
categories:
  - security
  - AI
tags:
  - aws
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
# Understanding OAuth Authentication in Amazon Bedrock AgentCore: A Deep Dive

## Introduction

Amazon Bedrock AgentCore introduces a sophisticated authentication pattern that enables AI agents to securely access external services on behalf of users. This architecture implements a **dual authentication pattern** that separates inbound authentication (who can call your agent) from outbound authentication (how your agent accesses external services).

In this post, we'll explore how this OAuth flow works, why it's designed this way, and walk through a complete authentication cycle with detailed diagrams.

## The Dual Authentication Pattern

Traditional applications typically implement authentication in one direction: users authenticate to access the application. But AI agents introduce a new challenge: the agent itself needs to authenticate to external services **on behalf of the user**.

Bedrock AgentCore solves this with two separate authentication layers:

### 1. Inbound Authentication (User → Agent)
Controls **who** can invoke your agent runtime. Uses JWT tokens validated against a Cognito User Pool.

### 2. Outbound Authentication (Agent → External Services)
Controls **how** your agent accesses external services. Uses OAuth 2.0 with user federation to act on behalf of the authenticated user.

<div class="mermaid">
graph LR
    subgraph Inbound["Inbound Authentication"]
        direction TB
        U[User] -->|JWT Token| R[Agent Runtime]
        R -->|Validates against| C1[Runtime Cognito Pool]
        R -.->|Executes| A[Agent Code] 
    end
    Inbound ~~~ Outbound

    subgraph Outbound["Outbound Authentication"]
        direction TB
        A -->|Needs external access| TV[Token Vault]
        TV -->|OAuth 2.0| C2[Identity Provider]
        C2 -->|Returns token| TV
        TV -->|Cached token| A
    end

    style U fill:#4dabf7
    style R fill:#51cf66
    style A fill:#fab005
    style TV fill:#7950f2
    style C1 fill:#4dabf7
    style C2 fill:#fab005
</div>

## Architecture Overview

### Two Cognito Pools: Why?

At first glance, using two separate Cognito User Pools might seem like unnecessary complexity. However, this architectural decision is fundamental to implementing secure, scalable AI agents that can access external services on behalf of users. The key insight is that authenticating who can invoke your agent is conceptually different from authenticating which external services your agent can access. By separating these concerns into two distinct authentication layers, we achieve better security isolation, clearer audit trails, and the flexibility to integrate with multiple identity providers without coupling them together. Think of it as having two separate security checkpoints: one at the entrance to your building (who can use the agent) and another at specific rooms inside (which external services the agent can access).

The architecture uses two separate Cognito User Pools, each serving a distinct purpose:

**Runtime Pool (Inbound Authentication):**
- **Purpose**: Authenticate callers who invoke the agent
- **Users**: Application users 
- **Flow**: User → JWT Token → Runtime validates token
- **Think of it as**: "Who can talk to my agent?"

**Identity Pool (Outbound Authentication):**
- **Purpose**: Store credentials for external services
- **Users**: Service accounts 
- **Flow**: Agent → Request token → External service validates
- **Think of it as**: "What can my agent access?"

<div class="mermaid">
graph TB
    subgraph "InboundAuth Cognito Pool"
        R1[Cognito Pool - InboundAuth]
        R2[Authenticates: Callers]
        R4[Validates: Inbound JWT tokens]
    end

    subgraph "OutboundAuth Cognito Pool"
        I1[Cognito Pool - OutboundAuth]
        I2[Authenticates: External Services]
        I4[Provides: OAuth tokens for agents]
    end

    User[End User] -->|Authenticate| R1
    R1 -->|JWT| Runtime[Agent Runtime]
    Runtime -->|Execute| Agent[Agent Code]
    Agent -->|Need token| Vault[Token Vault]
    Vault -->|OAuth flow| I1
    I1 -->|Access token| Vault

    style R1 fill:#4dabf7
    style I1 fill:#fab005
    style Runtime fill:#51cf66
</div>

This separation provides several benefits:
1. **Security**: Compromising user credentials doesn't expose service credentials providing isolation
2. **Scalability**: Different user pools can scale independently of each other 
3. **Flexibility**: Can integrate multiple external identity providers if you choose to do so
4. **Audit**: Clear separation between user actions and service actions

## The Complete OAuth Flow

Let's walk through a complete authentication cycle, from initial invocation to cached token usage.

<div class="mermaid">
sequenceDiagram
    participant User
    participant Browser
    participant CLI as AgentCore CLI
    participant Runtime as Bedrock AgentCore<br/>Runtime
    participant Agent as Agent Code<br/>(agent.py)
    participant TokenVault as Identity Token Vault
    participant RuntimeCognito as Cognito Pool - InboundAuth
    participant IdentityCognito as Cognito Pool - OutboundAuth

    rect rgb(200, 230, 255)
        Note over User,RuntimeCognito: PHASE 1: INBOUND AUTHENTICATION
        User->>CLI: 1. Get authentication token
        CLI->>RuntimeCognito: 2. Username/Password
        RuntimeCognito-->>CLI: 3. JWT Access Token
        Note right of CLI: Token contains:<br/>• client_id<br/>• username<br/>• scopes<br/>• expiry
        CLI->>User: 4. Return JWT token
    end

    rect rgb(255, 230, 200)
        Note over User,Runtime: PHASE 2: INVOKE AGENT
        User->>CLI: 5. Invoke agent with JWT
        CLI->>Runtime: 6. InvokeAgentRuntime API call
        Runtime->>RuntimeCognito: 7. Validate JWT against OIDC discovery
        RuntimeCognito-->>Runtime: 8. ✅ Token Valid
        Runtime->>Agent: 9. Execute agent code
    end

    rect rgb(230, 255, 230)
        Note over Agent,IdentityCognito: PHASE 3A: OUTBOUND AUTH - FIRST CALL
        Agent->>Agent: 10. Function with @requires_access_token
        Note right of Agent: Decorator parameters:<br/>• provider_name<br/>• callback_url<br/>• auth_flow: USER_FEDERATION<br/>• scopes: [openid]

        Agent->>TokenVault: 11. Request token for provider
        TokenVault->>TokenVault: 12. Check cache → Not found
        TokenVault->>IdentityCognito: 13. Initiate OAuth authorization
        IdentityCognito-->>TokenVault: 14. Authorization URL
        TokenVault-->>Agent: 15. Return auth URL
        Agent->>Agent: 16. on_auth_url callback fires
        Agent-->>Runtime: 17. Response: "Authorization Required"
        Runtime-->>CLI: 18. Forward response
        CLI-->>User: 19. Display authorization URL
    end

    rect rgb(255, 240, 230)
        Note over User,IdentityCognito: PHASE 3B: USER AUTHORIZATION
        User->>Browser: 20. Opens authorization URL
        Browser->>TokenVault: 21. GET authorization endpoint
        TokenVault->>IdentityCognito: 22. Redirect to Cognito login
        User->>Browser: 23. Enter credentials
        Browser->>IdentityCognito: 24. Submit authentication
        IdentityCognito->>IdentityCognito: 25. Validate credentials
        IdentityCognito->>IdentityCognito: 26. Generate authorization code
        IdentityCognito->>Browser: 27. Redirect with auth code
        Browser->>TokenVault: 28. Callback with code
    end

    rect rgb(240, 230, 255)
        Note over TokenVault,IdentityCognito: PHASE 3C: TOKEN EXCHANGE
        TokenVault->>IdentityCognito: 29. Exchange code for tokens
        Note right of TokenVault: Grant Type: authorization_code<br/>Client credentials included
        IdentityCognito-->>TokenVault: 30. Return tokens
        Note right of IdentityCognito: Returns:<br/>• access_token<br/>• id_token<br/>• refresh_token
        TokenVault->>TokenVault: 31. Cache tokens by session_id
        TokenVault->>Browser: 32. Redirect to callback_url
    end

    rect rgb(230, 255, 255)
        Note over User,IdentityCognito: PHASE 4: SUBSEQUENT CALLS (Cached)
        User->>CLI: 33. Invoke agent again (same session)
        CLI->>Runtime: 34. InvokeAgentRuntime with JWT
        Runtime->>Runtime: 35. Validate JWT (cached)
        Runtime->>Agent: 36. Execute agent code
        Agent->>TokenVault: 37. Request token
        TokenVault->>TokenVault: 38. Check cache → ✅ Found!
        TokenVault-->>Agent: 39. Return cached access_token
        Note right of Agent: Decorator injects token<br/>directly into function
        Agent->>Agent: 40. Execute function with token
        Agent-->>Runtime: 41. Success response
        Runtime-->>CLI: 42. Forward response
        CLI-->>User: 43. Display result
    end
</div>

### Understanding the Flow: A Simplified Walkthrough

The sequence diagram above shows the complete technical flow, but let's break it down into simple, digestible steps. Think of this as a story with four chapters: getting your ticket to use the agent, using the agent, getting permission for external access, and then enjoying fast subsequent access.

#### **Chapter 1: Getting Your Ticket to Talk to the Agent** (Steps 1-4)

Before you can ask your agent to do anything, you need to prove who you are. This is the **inbound authentication** step.

**Step 1: You ask for credentials**
You run a command like `agentcore identity get-cognito-inbound-token`. Think of this as walking up to a ticket booth and asking for admission.

**Step 2: System checks your identity**
Your username and password are sent to the Runtime Cognito Pool. This is like showing your ID to the ticket seller.

**Step 3: System gives you a JWT token**
If your credentials are valid, you receive a JWT (JSON Web Token). This token is like a concert ticket or an all-access pass - it proves you're allowed to invoke the agent. The token contains important information:
- Your client ID (which application you're using)
- Your username (who you are)
- Scopes (what you're allowed to do)
- Expiry time (typically 1 hour)

**Step 4: You hold onto your ticket**
You'll use this JWT token every time you talk to the agent.  you'll need it for every invocation.

---

#### **Chapter 2: Talking to the Agent** (Steps 5-9)

Now that you have your JWT ticket, you can actually invoke the agent. This is still part of **inbound authentication** - proving you have the right to use the agent.

**Step 5: You show your ticket and make a request**
You run: `agentcore invoke '{"prompt": "Check my external account"}' --bearer-token <JWT>`
You're essentially saying: "Here's my ticket, please do this task for me."

**Step 6: Ticket gets validated**
The Bedrock AgentCore Runtime receives your JWT and needs to verify it's legitimate. Just like a bouncer at a concert scanning your ticket.

**Step 7: Runtime calls the ticket office**
The Runtime asks the Runtime Cognito Pool: "Is this JWT token real? Is it still valid? Has it expired?"
This happens by checking against the OIDC discovery endpoint configured in your agent.

**Step 8: Cognito confirms**
✅ "Yes, this token is valid. This user is authorized to invoke the agent."
The signature is valid, the token hasn't expired, and it was issued by the correct authority.

**Step 9: Agent begins execution**
With authentication confirmed, the agent code starts executing with your request. Your prompt is passed to the agent, and it begins processing.

---

#### **Chapter 3A: Agent Needs External Access - First Time** (Steps 10-19)

Now we switch to **outbound authentication**. Your agent needs to access an external service on your behalf, but it doesn't have permission yet.

**Step 10: Agent encounters a protected function**
Your agent code calls a function decorated with `@requires_access_token`. This decorator is the key to the OAuth flow.

**Step 11: Agent asks the Token Vault**
The decorator automatically asks: "Do I have an access token for this external service provider for this session?"
The Token Vault is a secure storage system that caches OAuth tokens by session ID.

**Step 12: Vault checks its cache**
The Token Vault looks up: Session ID → Provider → Token
Result: ❌ "No token found. This is the first time this session is accessing this provider."

**Step 13: Vault initiates OAuth flow**
Since there's no cached token, the Token Vault starts the OAuth 2.0 authorization code flow with the Identity Cognito Pool.

**Step 14: External service creates authorization URL**
The Identity Provider (Identity Cognito) generates a special authorization URL. This URL contains:
- Encrypted state (including your session ID)
- Requested scopes (what permissions you're asking for)
- Callback URL (where to redirect after authorization)
- Client ID (which application is requesting access)

**Step 15: Vault returns the authorization URL to the agent**
Instead of a token, the Vault returns: "Authorization required - here's the URL"

**Step 16: Agent's callback hook fires**
The `on_auth_url` callback you specified in the decorator triggers. This gives your code a chance to handle the authorization URL appropriately.

**Step 17-19: Agent tells you authorization is needed**
The agent responds with a message like:
*"🔐 Authorization Required - Please open this URL in your browser to authorize: [URL]"*
The Runtime forwards this response, and you see it in your CLI. The ball is now in your court - you need to authorize the access.

---

#### **Chapter 3B: You Grant Permission** (Steps 20-28)

This is where **you** (the user) explicitly grant permission for the agent to access the external service on your behalf. This is the heart of USER_FEDERATION - you're in control.

**Step 20: You open the authorization URL**
You click the link (or copy-paste it into your browser). This opens the OAuth authorization flow in your web browser.

**Step 21: Browser navigates to the authorization endpoint**
Your browser makes a GET request to the Token Vault's authorization endpoint.

**Step 22: Redirected to the login page**
The Token Vault redirects you to the Identity Cognito Pool login page. This is where you'll authenticate to the external service.

**Step 23: You enter your credentials**
You type in your username and password for the external service. In this demo, that's the Identity Cognito user (like `externaluser24a901fd`).
**Important**: These are different credentials from your Runtime Cognito credentials! You're now proving you own the external account.

**Step 24: You submit the login form**
Browser sends your credentials to the Identity Cognito Pool.

**Step 25: Identity Cognito validates your credentials**
The external service checks: "Is this the correct password for this user?" If valid, it proceeds.

**Step 26: Identity Cognito generates an authorization code**
Instead of giving you the actual access token directly, OAuth uses an intermediate step: an authorization code. This is a short-lived, one-time-use code that can be exchanged for tokens.
**Why a code?** Security! The code is sent via the browser (less secure channel), but the actual tokens are exchanged server-to-server (more secure).

**Step 27: Browser redirected with the authorization code**
Identity Cognito redirects your browser back to the Token Vault callback URL, including the authorization code in the URL parameters.

**Step 28: Token Vault receives the code**
The Token Vault's callback endpoint receives the authorization code. Now it's ready for the final exchange.

---

#### **Chapter 3C: Authorization Code Becomes Real Access** (Steps 29-32)

The Token Vault now exchanges the temporary authorization code for real, usable access tokens. This happens server-to-server, away from the browser.

**Step 29: Vault exchanges code for tokens**
The Token Vault makes a server-to-server call to Identity Cognito:
*"Here's the authorization code. Please give me access tokens. Here's my client secret to prove I'm authorized."*

This exchange uses the `authorization_code` grant type and includes:
- The authorization code (from step 27)
- Client ID (identifies your application)
- Client secret (proves your application is legitimate)
- Redirect URI (must match the original request)

**Step 30: Identity Cognito returns three tokens**
The Identity Provider responds with a token bundle:

1. **access_token**: This is the golden ticket! Your agent uses this to make API calls to the external service. It's typically valid for 1 hour.

2. **id_token**: A JWT containing claims about the user's identity (who they are, when they logged in, etc.). Useful for displaying user information.

3. **refresh_token**: A long-lived token used to obtain new access tokens when they expire. The agent can use this automatically to refresh access without asking you to re-authorize.

**Step 31: Vault caches the tokens**
The Token Vault stores all three tokens in its cache, indexed by:
- Session ID (e.g., `demo_session_ABC123`)
- Provider name (e.g., `ExternalServiceProvider`)

This cache means future requests in the same session won't need re-authorization!

**Step 32: Browser redirected to callback URL**
Your browser is redirected to the `callback_url` specified in the decorator (e.g., `https://example.com/oauth/callback`).
In this demo, it's a dummy URL that does nothing. In production, this would be your application's URL that handles post-authorization logic (like showing a success message or closing the auth window).

---

#### **Chapter 4: Subsequent Calls Are Lightning Fast!** (Steps 33-43)

This is where you see the real benefit of OAuth token caching. The second time you invoke the agent in the same session, everything is already set up.

**Step 33: You invoke the agent again**
You run the same command: `agentcore invoke '{"prompt": "Check my account"}' --bearer-token <JWT>`
Crucially, you're using the **same session ID** as before.

**Step 34: JWT validation (same as before)**
The Runtime still validates your JWT token - you still need to prove you're authorized to invoke the agent.

**Step 35: Validation is cached/fast**
The Runtime may have cached the JWT validation results, making this step very quick.

**Step 36: Agent code executes**
Your agent code runs, and again encounters the function with `@requires_access_token`.

**Step 37: Agent asks Token Vault for the token**
The decorator asks: "Do I have an access token for this provider and session?"

**Step 38: Vault checks cache - SUCCESS!**
✅ The Token Vault finds the cached token from Phase 3C:
*"Found it! Session `demo_session_ABC123` → Provider `ExternalServiceProvider` → access_token: `eyJraWQi...`"*

**Step 39: Vault returns the cached access token**
The Token Vault immediately returns the access token. No authorization URL, no user interaction needed!

**Step 40: Function executes with the token**
The decorator automatically injects the token into your function's `access_token` parameter. Your function code runs with the token available:

```python
async def get_identity_token(*, access_token: str) -> str:
    # access_token is already here! No OAuth flow needed!
    return access_token
```

**Step 41: Agent completes successfully**
Your agent logic executes, possibly making API calls to the external service using the access token. It returns a success response.

**Step 42: Runtime forwards the response**
The Bedrock AgentCore Runtime sends the response back to the CLI.

**Step 43: You see the result**
You see: *"✅ Authenticated to external service. Token length: 847 characters. Status: Active and cached for this session"*

The entire flow from step 33 to 43 takes just milliseconds because everything is cached!

---


### Why This Two-Step Process?

Now that we've walked through all four chapters of the OAuth flow, you can see how the architecture elegantly handles both authentication challenges. The first time through requires user interaction and multiple network calls, taking several seconds to complete. But subsequent invocations in the same session are blazingly fast because everything is cached - the JWT validation is quick, and the OAuth token is retrieved from memory rather than requiring another authorization flow. This pattern strikes a perfect balance between security (explicit user authorization) and user experience (fast, seamless subsequent operations). The question naturally arises: why go through this two-step process at all? Why not use a single token for everything? The answer lies in the fundamental separation of concerns between who can invoke your agent versus what your agent can access on your behalf.


1. **Security Isolation**: Compromising your inbound JWT doesn't expose your external service credentials, and vice versa.

2. **Different Lifetimes**: Your JWT and OAuth tokens can expire independently and be refreshed separately.

3. **Principle of Least Privilege**: The agent only gets access to external services when you explicitly grant it.

4. **Auditability**: Clear separation between "who invoked the agent" (JWT) and "what external services were accessed" (OAuth).

5. **Flexibility**: You can revoke external service access without revoking agent access, or the other way around.

### Why Cache Tokens?

Another design decision that might seem obvious in retrospect but is critical to understand is the token caching mechanism. Without caching, every single agent invocation would require a complete OAuth authorization flow - you'd need to click an authorization link, log in, and grant permission every time you ask your agent a simple question. This would make the system practically unusable. Token caching solves this by storing the OAuth access tokens (and refresh tokens) in memory, indexed by session ID and provider name. When your agent needs to access an external service, it first checks the cache: if a valid token exists, it's used immediately; if not, the OAuth flow kicks in. This approach transforms the user experience from "authorize every request" to "authorize once per session," while maintaining security through session isolation and token expiration. Let's examine why this caching strategy is so important:

The caching in Phase 4 is crucial for user experience:

- **Performance**: Token exchange is slow (involves multiple network calls and redirects). Caching makes subsequent calls 10-100x faster.

- **User Experience**: Imagine having to click an authorization link every single time you ask your agent a question! Caching means you authorize once per session.

- **Rate Limiting**: Many OAuth providers have rate limits on token exchanges. Caching reduces the number of authorization flows.

- **Security**: Tokens are cached per session ID, ensuring isolation between different users and contexts.

### Session Isolation: Why It Matters

A subtle but powerful aspect of the token caching architecture is session-based isolation. You might wonder: why not cache tokens globally per user, so that once you authorize, all future agent invocations by that user across any session can use the same token? While this would be more convenient, it would also create significant security risks. By tying tokens to specific session IDs rather than user identities, the system ensures that each invocation context is isolated from others. This means that if a session is compromised, only that session's tokens are at risk - not all of the user's access across all sessions. It also enables fine-grained control: you can revoke access for a specific session without affecting other active sessions, and audit logs can precisely track which session performed which action. Session isolation is the foundation that makes the entire caching mechanism both performant and secure.

### Session-Based Token Caching

Token caching is tied to session IDs, not user IDs. This provides important security and isolation benefits:

<div class="mermaid">
graph TB
    subgraph "Session A: demo_session_ABC123"
        A1[First Invocation] -->|No token| A2[User authorizes]
        A2 --> A3[Token cached for Session A]
        A3 --> A4[Second Invocation]
        A4 -->|Token found| A5[✅ Use cached token]
        A5 --> A6[Subsequent calls fast]
    end

    subgraph "Session B: demo_session_XYZ789"
        B1[First Invocation] -->|Different session<br/>No token| B2[User must authorize again]
        B2 --> B3[Token cached for Session B]
        B3 --> B4[Isolated from Session A]
    end

    subgraph "Token Vault Cache"
        Cache[Session → Token Map]
        Cache -.->|Lookup| A3
        Cache -.->|Lookup| B3
    end

    style A3 fill:#51cf66
    style A5 fill:#51cf66
    style B4 fill:#fab005
</div>

Notice that tokens are tied to your session ID (e.g., `demo_session_ABC123`). This is a critical security feature:

- **Different session = Different tokens**: If you start a new session (new session ID), you'll need to re-authorize. The previous session's tokens aren't accessible.

- **Multi-user safety**: In a shared environment, User A's tokens never leak to User B because they have different session IDs.

- **Granular control**: You can invalidate a single session's access without affecting other sessions.

- **Audit trail**: Every action is tied to a specific session, making it easy to trace who did what.

## Conclusion

AWS Bedrock AgentCore's dual authentication pattern represents a thoughtful approach to one of the most challenging problems in AI agent development: how to enable agents to securely access external services on behalf of users while maintaining strong security boundaries, excellent user experience, and clear auditability. By separating inbound authentication (who can invoke your agent) from outbound authentication (what external services your agent can access), the architecture achieves the right balance between security and usability. 

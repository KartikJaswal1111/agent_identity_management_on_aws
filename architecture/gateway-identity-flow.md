# Sequence - One Request Through Gateway, Identity, and AVP

This traces a single chat message ("show me current inventory levels") through every identity
layer added across Activities 2-5, using the Activity 4 (Inventory, cross-trust-domain) path since
it exercises all three primitives at once: inbound Cognito JWT auth on the agent runtime, an
AgentCore Identity outbound token exchange, and an AVP authorization check gating whether the
Inventory tool group is even loaded for this caller.

```mermaid
sequenceDiagram
    autonumber
    participant User as Authenticated user (e.g. lisa.rodriguez, Manager)
    participant Runtime as AgentCore Runtime (Cognito JWT authorizer)
    participant Agent as agent_core.py (Strands Agent)
    participant AVP as Amazon Verified Permissions (Cedar policy store)
    participant Identity as AgentCore Identity (OAuth2 credential provider)
    participant Gateway as AgentCore Gateway (Inventory)
    participant MCP as Inventory vendor's MCP server (separate trust domain)

    User->>Runtime: Chat message + Cognito ID token
    Runtime->>Runtime: Validate JWT (customJWTAuthorizer)
    Runtime->>Agent: invoke(payload, context) with principal token forwarded
    Agent->>Agent: build_session_tools(context, stack)
    Agent->>AVP: batch_is_authorized_with_token(declared resource ids, principal token)
    AVP-->>Agent: ALLOW for "inventory-tool-group" (Manager policy permits it)
    Note over Agent: Phase 1 pre-check passed - safe to open the connection
    Agent->>Identity: requires_access_token(provider=inventory-oauth-client-mcp-server, auth_flow=M2M)
    Identity->>Identity: Exchange agent workload identity for a short-lived OAuth2 access token
    Identity-->>Agent: access_token (Bearer)
    Agent->>Gateway: MCP list_tools (Authorization: Bearer access_token)
    Gateway-->>Agent: tool list (e.g. get-inventory___query)
    Agent->>AVP: batch_is_authorized_with_token(actual tool prefixes, principal token)
    AVP-->>Agent: ALLOW get-inventory
    Note over Agent: Phase 2 authoritative check - only allowed tools kept
    Agent->>Gateway: MCP call_tool(get-inventory, product_id=...)
    Gateway->>MCP: Forward call (vendor's own AuthN on their side)
    MCP-->>Gateway: Inventory data
    Gateway-->>Agent: Tool result
    Agent-->>Runtime: Formatted Markdown table response
    Runtime-->>User: "Here's current inventory..."
```

## What each layer is actually deciding

- **Runtime JWT validation (steps 1-2)** answers *"is this a real, authenticated AnyCompany user at
  all?"* - unrelated to which tools they can use.
- **AVP phase 1 (steps 5-6)** answers *"is it even worth opening a connection to this tool group for
  this caller?"* - a cheap skip, not the authoritative decision.
- **AgentCore Identity (steps 8-10)** answers *"what credential does the agent present to a system
  outside AnyCompany's own trust domain?"* - the M2M token exchange means the Inventory vendor's
  OAuth2 client secret never touches the agent's code or container image (F2/F4 from the HLD).
- **AVP phase 2 (steps 12-13)** is the *authoritative* decision, made after the real tool list comes
  back - this is what actually determines which tool objects the LLM ever sees.

A different caller (e.g. james.miller, an external Supplier) hits the same sequence but gets an
AVP `DENY` at step 6 for tool groups outside their role - the connection to that gateway is never
even opened, which is why [Activity 4](../docs/04-activity4-add-the-inventory-mcp-tool.md) and
[Activity 5](../docs/05-activity5-dynamic-tool-filtering-capability.md) show different users of the identical
agent seeing entirely different tool sets.

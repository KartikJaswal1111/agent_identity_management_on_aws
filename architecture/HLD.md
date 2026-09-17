# High-Level Design - Identity for an AI Agent's Tools

## The problem this is actually solving

An AI agent that can call tools is, functionally, a new kind of API caller - one that a human isn't
directly supervising on every call. Once an agent can reach a company's sales data, product catalog,
customer reviews, and a supplier's inventory system, the question stops being "can the agent call
this API" and becomes three separate, harder questions:

1. **Whose identity is the agent acting as** when it calls a given tool - its own service identity,
   or the identity of the human it's currently helping?
2. **How does a credential ever cross a trust boundary** - e.g. AnyCompany's agent needing to call a
   *supplier's* inventory MCP server, where AnyCompany has no account in the supplier's domain?
3. **Does every user of the agent see the same toolset**, or should a supplier's rep, a store
   manager, and an admin get different tools without three different agents to maintain?

This project is the requirements-driven answer to those three questions, built incrementally across
six activities in the SEC307 workshop, each one adding exactly one new identity primitive.

## Requirements

### Functional
- **F1** - The agent must call backend tools (Lambda, API Gateway) without the tool owner having to
  trust the agent's raw IAM role directly.
- **F2** - The agent must be able to call a tool hosted in a *different* AWS account/organization
  (the Inventory vendor) without ever holding that vendor's long-lived credentials.
- **F3** - Different authenticated users of the same agent must see different subsets of tools,
  driven by their role - without branching the agent's code per role.
- **F4** - No client secret should live in the agent's code or container image.

### Non-functional
- **NF1 (Least privilege)** - A tool's target should only be reachable by callers explicitly
  authorized for it, not merely "anyone who can reach the gateway."
- **NF2 (No standing secrets)** - Credentials for outbound calls should be short-lived and minted
  per-request, not stored.
- **NF3 (Auditability)** - An authorization decision (allow/deny a specific tool to a specific
  caller) should be attributable to an explicit, versioned policy, not implicit code logic.
- **NF4 (Incremental adoption)** - Each new tool/trust boundary should be addable without having to
  redesign the tools already wired in.

## Candidate approaches (per requirement, scored)

| Approach | F1 | F2 | F3 | F4 | NF1 | NF2 | NF3 |
|---|---|---|---|---|---|---|---|
| **A - Static API keys hardcoded per tool** | Partial | No | No | Fails | Fails | Fails | Fails |
| **B - One shared IAM role for all outbound calls** | Yes | No | No | Yes | Fails | Partial | Fails |
| **C - AgentCore Gateway (per-tool auth) + AgentCore Identity (outbound token vault) + Cedar/AVP (per-call authorization)** | Yes | Yes | Yes | Yes | Yes | Yes | Yes |

Approach A fails immediately on F2 and F4 and was never seriously on the table - it's listed only
because it's the default a non-identity-aware implementation would reach for. Approach B (what
`create_streamable_http_transport_oauth2` in `agent_core.py` is explicitly commented as being
*retained for backward compatibility* rather than the intended path) solves F1 but re-centralizes
trust: every tool call looks identical to an auditor, and there's no way to say "this caller may use
the Sales tool but not the Inventory tool" without branching code. It also can't cross the trust
boundary in F2 without static, vendor-issued secrets, which fails NF2.

**Recommendation: Approach C**, layered on incrementally exactly as the workshop's six activities
did:

| Activity | Primitive added | Requirement it satisfies |
|---|---|---|
| 2 | AgentCore Gateway, IAM (SigV4) inbound auth, single Lambda target | F1 |
| 3a/3b/3c | AgentCore Gateway, Cognito (OAuth2) inbound auth, API Gateway targets, multiple tools behind one gateway | F1, NF4 |
| 4 | AgentCore Identity outbound credential provider (M2M, cross-account) | F2, F4, NF2 |
| 5 | Amazon Verified Permissions (Cedar) as a per-tool authorization gate in front of tool loading | F3, NF1, NF3 |

The gateway/identity split is what makes this composable: a gateway only decides *who may reach
this endpoint at all* (inbound auth), while AgentCore Identity decides *what credential the agent
presents on the far side* (outbound auth) - two independent axes, so adding a new tool with a new
trust boundary (Activity 4) never touches how the existing tools authenticate.

## How authorization actually happens on each request

See [`gateway-identity-flow.md`](gateway-identity-flow.md) for the full sequence, but the shape of
it: `build_session_tools()` in `agent_core.py` runs a two-phase check against Verified Permissions
for every incoming request - first a cheap pre-check against each tool group's *declared* resource
ids (skip opening an MCP connection at all if the caller can't use anything in that group), then an
authoritative per-tool-prefix check once the real tool list comes back from the gateway (handles a
gateway that fronts more targets than the group declared). Nothing is filtered by editing the
agent's prompt or hoping the model declines - the tool objects themselves are removed from the list
handed to the LLM before it ever sees them.

## Decision drivers that would flip this recommendation

- If the number of tools were small and fixed (say, exactly one, forever), Approach B's simplicity
  would be defensible - the Gateway/Identity/AVP layering earns its complexity specifically because
  tools and trust boundaries kept growing across the six activities.
- If every caller of the agent had identical permissions (no manager/supplier/admin distinction),
  Activity 5's AVP layer would be pure overhead - F3 wouldn't exist as a requirement.

## What I'd change for a production system

- Version the Cedar policies (Terms of Service, Sales, Reviews, Admin, Inventory, plus whatever the
  sixth is) the same way an API is versioned, since `batch_is_authorized_with_token` calls are only
  as auditable as the policy version behind them at call time.
- Add a circuit breaker around `_authorize_tool_ids` - as written, an AVP outage with
  `AVP_POLICY_STORE_ID` set makes every tool group fail closed silently (an empty allowed set), which
  is the safe default but should page someone, not just quietly serve a smaller toolset.
- Turn the workshop's per-activity `agent.py.backup` file (written by every `activity*_gen.sh`
  before it edits `agent.py`) into a real version-controlled change instead of a single rolling
  backup, so a bad snippet substitution is a revert, not an overwrite.

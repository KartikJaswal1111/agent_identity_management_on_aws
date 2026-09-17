# Activity 4: Add the Inventory MCP Tool

## Architecture after this activity

```mermaid
graph LR
    subgraph TD1["Trust Domain 1 - AnyCompany"]
        User["Authenticated user"] -->|Cognito JWT| Runtime["AgentCore Runtime<br/>Strands Agent"]
        Runtime -->|IAM Auth| GW1["AgentCore Gateway<br/>AnyCompany-ToS-Tool"]
        Runtime -->|"JWT Auth (2LO)"| GW2["AgentCore Gateway<br/>AnyCompany-Sales-Products-Reviews-Tool"]
        Runtime -->|"JWT Auth (2LO)"| GW4["AgentCore Gateway<br/>AnyCompany-Inventory-Tool"]
        Identity["AgentCore Identity<br/>OAuth2 credential provider<br/>(vendor-issued client secret)"]
        Runtime -.->|mint M2M token for GW4| Identity
        GW1 --> ToS["Terms of Service Lambda"]
        GW2 -->|API Key| Sales["Sales API Gateway"]
        GW2 -->|"OAuth2 (2LO)"| Products["Products API Gateway"]
        GW2 -->|IAM Auth| Reviews["Customer Reviews DynamoDB"]
    end
    subgraph TD2["Trust Domain 2 - Inventory Vendor (external, own account)"]
        VendorCognito["Vendor's own<br/>Amazon Cognito"]
        VendorIdentity["Vendor's own<br/>AgentCore Identity"]
        GW3["Vendor's own AgentCore Gateway"]
        VendorCognito -.-> GW3
        VendorIdentity -.-> GW3
        GW3 -->|Lambda Authorizer| Inventory["Inventory<br/>API Gateway + Lambda"]
    end
    GW4 -->|"JWT Auth (2LO), second hop"| GW3
```

The first boxes outside AnyCompany's own trust domain. What's easy to miss from the agent's code
alone: **there are two gateways involved, not one.** `AnyCompany-Inventory-Tool` (`GW4` above) is a
gateway AnyCompany itself owns and configures, sitting in Trust Domain 1 like every other gateway -
the agent only ever calls *this* one directly, using an M2M token from its own AgentCore Identity.
That gateway's own *target*, though, is configured (at the AWS console level, not in
`agent_core.py`) to point at the vendor's own, independently-operated AgentCore Gateway in Trust
Domain 2 - which has its own Cognito user pool and its own AgentCore Identity, entirely separate
from AnyCompany's. See [`architecture/gateway-identity-flow.md`](../architecture/gateway-identity-flow.md)
for the request sequence from the agent's point of view, and
[`architecture/system-architecture.md`](../architecture/system-architecture.md) for the full
two-gateway topology.

## A tool AnyCompany doesn't own

Every tool so far lives inside AnyCompany's own AWS account. Activity 4 adds a tool that doesn't:
an Inventory MCP server run by an external supplier, in a trust domain AnyCompany has no account
in. This is the activity that actually needs [`code/agent_core.py`](../code/agent_core.py)'s third
transport function, `create_streamable_http_transport_agentcore_identity` - the other two
(SigV4, OAuth2-with-embedded-secret) can't cross an account boundary without a shared, standing
credential.

## Setting up the outbound OAuth2 client

An OAuth2 client for the Inventory vendor's MCP server is created in Cognito, but its secret is
handed to **AgentCore Identity**, not the agent:

![Add OAuth Client: inventory-oauth-client-mcp-server](images/activity4-01-add-oauth-client-inventory.png)
![OAuth client provider configuration detail](images/activity4-02-add-oauth-client-inventory-cognito-provider.png)

`activity4_gen.sh` (see [`code/README.md`](../code/README.md)) is explicit in its own comments
about why: *"this script NO LONGER fetches or embeds the Cognito client secret... the agent
exchanges its workload identity for a token at runtime (M2M)."* Only the non-secret provider name
and gateway URL get injected into `agent_core.py` - the secret itself never leaves AgentCore
Identity's vault.

## A new gateway for a new trust boundary

Unlike Activity 3's targets, which shared one gateway, Inventory gets its own - `AnyCompany-
Inventory-Tool`, created here in AnyCompany's own account:

![Create gateway: AnyCompany-Inventory-Tool](images/activity4-03-create-gateway-inventory-tool.png)
![Review and create: Inventory gateway IAM role permissions](images/activity4-04-review-create-inventory-gateway.png)

This gateway's *target*, though, isn't a Lambda or an API Gateway in AnyCompany's own account the
way every earlier target was - it's configured to reach the Inventory vendor's own, separately
operated AgentCore Gateway, sitting behind the vendor's own Cognito user pool and their own
AgentCore Identity. That second hop (`AnyCompany-Inventory-Tool` → the vendor's gateway) is itself
authenticated with a second, independent OAuth2 (2LO) exchange, using credentials the vendor issued
to AnyCompany - which is exactly what the OAuth2 client set up above is for.

## The M2M token exchange at runtime

`agent_core.py`'s `create_streamable_http_transport_agentcore_identity` wraps a function decorated
with `@requires_access_token(..., auth_flow="M2M")` from `bedrock_agentcore.identity.auth` - at
connection time, the agent exchanges its own workload identity for a short-lived access token
scoped to the Inventory provider, then attaches it as a Bearer token on the MCP connection to
`AnyCompany-Inventory-Tool` (`GW4`). Everything past that gateway - the second hop into the
vendor's own domain - happens inside AWS's gateway target configuration, invisible to this
function entirely. Because
this callback runs inside Strands' async background thread, `agent_core.py` also has to route it
through `_run_coroutine_blocking` to avoid an `asyncio.run() cannot be called from a running event
loop` error - a detail visible only in the source, not in any console screen, but worth calling out
since it's the kind of runtime-context bug that only surfaces under Strands' specific threading
model.

## Testing across the trust boundary

With the token exchange wired in, the agent can query the external vendor's inventory as if it
were a local tool - the trust boundary is invisible from the chat interface:

![Chat test: inventory levels query](images/activity4-05-agent-inventory-levels-query.png)
![Chat test: cross-tool query combining sales and inventory data](images/activity4-06-agent-best-selling-products-inventory.png)

The second test is the more interesting one: "most selling products" pulls from *both* the Sales
tool (Activity 3a, same-account) and the Inventory tool (this activity, cross-account) in a single
response, with the agent never needing to know or care that the two tools authenticate completely
differently under the hood.

## What this activity proves

F2 and F4 from the [HLD](../architecture/HLD.md#requirements) - crossing a trust boundary, and
never storing a client secret in the agent's own code or image - are satisfied by the same
mechanism: AgentCore Identity as a credential vault the agent's workload identity can exchange
against, per request, rather than a secret baked into a config file. See
[`architecture/gateway-identity-flow.md`](../architecture/gateway-identity-flow.md) for the full
request sequence through this exact path. The next question - whether *every* authenticated user
should be able to reach this tool at all - isn't answered until
[Activity 5](05-activity5-dynamic-tool-filtering-capability.md).

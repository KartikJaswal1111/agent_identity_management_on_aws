# Activity 3a: Deploy the Sales tool

Part of [Activity 3: Deploy the Sales, Products, and Reviews tools](03-activity3-sales-products-reviews-tools.md).

## Architecture after this activity

```mermaid
graph LR
    subgraph TD1["Trust Domain 1 - AnyCompany"]
        User["Authenticated user"] -->|Cognito JWT| Runtime["AgentCore Runtime<br/>Strands Agent"]
        Runtime -->|"JWT Auth (2LO)"| GW2["AgentCore Gateway<br/>AnyCompany-Sales-Products-Reviews-Tool"]
        GW2 -->|API Key| Sales["Sales<br/>API Gateway + Lambda"]
        GW1["AgentCore Gateway<br/>AnyCompany-ToS-Tool"]
        Runtime -.-> GW1
    end
```

Note the two auth modes in play here aren't the same thing: the Runtime authenticates *to the
gateway* with a JWT (2LO) - that's Activity 3's whole gateway, uniform across every target added
in this and the next two sub-activities. The gateway then authenticates *to the Sales backend* with
a bare **API key** - a target-level auth choice independent of the gateway's own inbound mode. See
[Activity 3's overview](03-activity3-sales-products-reviews-tools.md#one-gateway-three-targets-three-different-backend-auth-modes)
for why Products and Customer Reviews use two more, different target-level auth modes on this same
gateway.

Only the Sales target is live so far - Products and Reviews are added in
[3b](03b-activity3b-deploy-the-products-tool.md) and [3c](03c-activity3c-deploy-the-customer-reviews-tool.md).

## Setting up the gateway and its outbound auth

The Sales Lambda's outputs are fetched from CloudFormation first:

![CloudFormation outputs including the Sales Lambda ARN](images/activity3a-01-cloudformation-outputs-lambda-arn.png)

AgentCore Identity's outbound auth list already shows OAuth2 credential providers for both
Inventory and Sales MCP clients, plus the shared Cognito OAuth2 provider - set up ahead of the
gateway itself:

![AgentCore Identity outbound auth: Cognito and per-tool OAuth2 clients](images/activity3a-02-outbound-auth-oauth-clients.png)

Worth flagging honestly: this console screen shows a `sales-mcp-oauth-client` OAuth2 provider
already provisioned, yet the workshop's own architecture diagrams and reference table both
consistently label the Sales target's actual auth as a plain **API key**, not OAuth2 (see
[Activity 3's overview](03-activity3-sales-products-reviews-tools.md)). The most likely
explanation is that the bootstrap stack pre-provisions credential providers for more than one
"choose your own adventure" path than any single participant actually exercises - this repo
follows the diagrams' API Key labeling below since that's what's shown wired to the deployed
target, but the unused OAuth2 provider is left visible here rather than quietly ignored.

Configuring inbound identity for the new gateway selects JWT this time, pointed at the Cognito
user pool:

![Configure Inbound Identity: JWT selected](images/activity3a-03-configure-inbound-identity-jwt.png)

The Cognito user pool (`sec307-agent-identity-corp-pool`) backing all of this already has five app
clients defined - one per tool plus a frontend client - visible in the console:

![Cognito App clients list, five real app clients](images/activity3a-04-cognito-app-clients-list.png)
![Cognito User Pool Overview with OIDC discovery URL](images/activity3a-05-cognito-user-pool-overview-arn.png)

The gateway's JWT authorizer config points at that pool's discovery URL directly:

![Gateway JWT schema configuration with real Discovery URL](images/activity3a-06-jwt-schema-configuration.png)

## Adding the Sales target

[`code/schemas/view-sales-api.json`](../code/schemas/view-sales-api.json) is the OpenAPI spec for
this target - an API Gateway endpoint, not a bare Lambda, which is why it needs a `servers.url`
filled in with the real execute-api endpoint before it can be used:

![view-sales-api.json schema in the IDE](images/activity3a-07-view-sales-api-json-schema.png)
![Editing the API Gateway URL into the schema before upload](images/activity3a-08-edit-sales-api-gateway-url.png)
![Add targets: Sales-API-Gateway target](images/activity3a-09-add-targets-sales-api-gateway.png)
![Inline schema editor showing the live API Gateway execute-api URL](images/activity3a-10-inline-schema-editor-sales-api.png)
![Review and create: Sales gateway summary](images/activity3a-11-review-create-sales-reviews-gateway.png)

## Wiring it into the agent and testing

`activity3a_gen.sh` wires the new gateway's URL into `agent_core.py`, the same splice pattern used
in [Activity 2](02-activity2-add-the-terms-of-service-tool.md):

![Running activity3a_gen.sh](images/activity3a-12-activity3a-gen-script-terminal.png)

Testing confirms the agent can now answer sales questions grounded in the live API:

![Chat test: RC car sales query, returned as a table](images/activity3a-13-agent-rc-car-sales-query.png)
![Chat test: best-selling analysis by product](images/activity3a-14-agent-sales-data-by-product.png)

Next: [Activity 3b - Deploy the Products tool](03b-activity3b-deploy-the-products-tool.md).

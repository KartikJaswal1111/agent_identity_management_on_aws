# Activity 2: Add the Terms of Service tool

## Architecture after this activity

```mermaid
graph LR
    subgraph TD1["Trust Domain 1 - AnyCompany"]
        User["Authenticated user"] -->|Cognito JWT| Runtime["AgentCore Runtime<br/>Strands Agent"]
        Runtime -->|IAM / SigV4| GW1["AgentCore Gateway<br/>AnyCompany-ToS-Tool"]
        GW1 --> ToS["Terms of Service Lambda"]
    end
```

The first gateway, fronting a single Lambda with the agent's own IAM execution role as the
credential - no OAuth2 client to manage yet, since there's nothing to federate.

## Adding the first identity boundary

Activity 2 is where AgentCore Gateway shows up for the first time: a single Lambda tool (Terms of
Service lookup) gets fronted by a gateway with **IAM/SigV4** inbound auth - the agent's own
execution role is the credential, no separate client secret to manage yet. Creating the gateway
starts with naming it and picking its inbound auth mode:

![Create gateway: AnyCompany-ToS-Tool](images/activity2-01-create-gateway-details.png)

The workshop's CloudFormation stacks (`bootstrap-stack`, `direct-targets-stack`,
`standalone-mcp-server-stack`) are what actually provisioned the Lambda and its schema behind the
scenes - the gateway just needs to know where to point:

![CloudFormation stacks provisioning the workshop's backend resources](images/activity2-02-cloudformation-stacks-list.png)

Step 2 of gateway creation is choosing the inbound identity mode - IAM for this activity, JWT from
Activity 3 onward:

![Configure Inbound Identity: IAM vs JWT choice](images/activity2-03-configure-inbound-identity.png)

## Wiring the target

Adding the Lambda as an MCP target requires its input/output schema - this is
[`code/schemas/get-tos-lambda.json`](../code/schemas/get-tos-lambda.json), a plain JSON Schema
describing a single `clause_type` parameter (`Refund`, `Delivery`, or `Payment`):

![Add targets: selecting the MCP target type](images/activity2-04-add-targets-mcp-target.png)
![Inline schema editor showing the ToS Lambda schema](images/activity2-05-inline-schema-editor-tos.png)

The final review step shows the gateway's IAM role permissions before creation:

![Review and create: gateway summary and IAM permissions](images/activity2-06-review-create-gateway-permissions.png)
![Gateways list showing the created gateway](images/activity2-07-gateways-list.png)

## Wiring the gateway into the agent

`activity2_gen.sh` is the first of the five generator scripts - it takes the gateway's MCP URL as
its one argument, normalizes it to end in `/mcp`, and splices a connection snippet between the
`START OF ACTIVITY 2` / `END OF ACTIVITY 2` markers in `agent_core.py`:

![Running activity2_gen.sh against the new gateway URL](images/activity2-08-activity2-gen-script-terminal.png)

The snippet it inserts registers a tool group using
`create_streamable_http_transport_sigv4` - the SigV4-signing transport in
[`code/streamable_http_sigv4.py`](../code/streamable_http_sigv4.py) - since this gateway's inbound
auth is IAM, not Cognito. Redeploying picks the change up the same way as Activity 1:

![Redeploying via launchAgent.sh after the agent.py update](images/activity2-09-launchagent-redeploy-start.png)
![Deployment success with ECR image URI](images/activity2-10-deployment-success-ecr-uri.png)
![Deployment details showing the updated Agent ARN](images/activity2-11-deployment-details-agent-arn.png)

## Testing the first grounded answer

With the Terms of Service tool live, the same "decline everything" agent from Module 1 can now
answer a real policy question, sourced entirely from the tool's response:

![Asking about delivery conditions](images/activity2-12-agent-delivery-conditions-query.png)
![Agent's grounded Standard Shipping response](images/activity2-13-agent-standard-shipping-response.png)

## What this activity establishes

The `TOOL_GROUPS` registry pattern in `agent_core.py` (see
[`code/README.md`](../code/README.md)) is introduced here in its simplest form: one group, one
connection function, no resource ids yet (AVP filtering doesn't exist until
[Activity 5](05-activity5-dynamic-tool-filtering-capability.md)). Every later activity adds a group to the same
list rather than restructuring how tools are loaded - see
[Activity 3](03-activity3-sales-products-reviews-tools.md) for the first gateway that fronts more than one tool
at once.

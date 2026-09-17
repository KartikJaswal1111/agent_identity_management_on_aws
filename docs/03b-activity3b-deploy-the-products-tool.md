# Activity 3b: Deploy the Products tool

Part of [Activity 3: Deploy the Sales, Products, and Reviews tools](03-activity3-sales-products-reviews-tools.md).
Follows [Activity 3a - Sales](03a-activity3a-deploy-the-sales-tool.md).

## Architecture after this activity

```mermaid
graph LR
    subgraph TD1["Trust Domain 1 - AnyCompany"]
        User["Authenticated user"] -->|Cognito JWT| Runtime["AgentCore Runtime<br/>Strands Agent"]
        Runtime -->|"JWT Auth (2LO)"| GW2["AgentCore Gateway<br/>AnyCompany-Sales-Products-Reviews-Tool"]
        GW2 -->|API Key| Sales["Sales<br/>API Gateway + Lambda"]
        GW2 -->|"OAuth2 (2LO) via<br/>Lambda Authorizer"| Products["Products<br/>API Gateway + Lambda"]
        GW1["AgentCore Gateway<br/>AnyCompany-ToS-Tool"]
        Runtime -.-> GW1
    end
```

Sales and Products are both live now - Reviews is added next in
[3c](03c-activity3c-deploy-the-customer-reviews-tool.md). Unlike Sales's bare API key, Products'
target auth is a genuine OAuth2 client-credentials flow, enforced by a **Lambda Authorizer** sitting
in front of the Products API Gateway - the authorizer is what actually validates the OAuth2 token
before the request reaches the Lambda, rather than API Gateway's native auth options.

## A new OAuth2 client for a new target

Each new tool in this gateway gets its own OAuth2 app client where its backend calls for one - the
instructions walk through creating a client-credentials secret specific to the Products tool:

![Instructions: creating the Products OAuth client secret](images/activity3b-01-oauth-client-secret-instructions.png)
![Cognito User Pool Overview, Products client context](images/activity3b-02-cognito-user-pool-overview.png)
![Add OAuth Client: products-oauth-client-resource-server](images/activity3b-03-add-oauth-client-products.png)

## Adding the Products target

[`code/schemas/view-product-catalog-api.json`](../code/schemas/view-product-catalog-api.json)
supports filtering by brand and product name (see the `curl` examples in
[`code/schemas/README.md`](../code/schemas/README.md)):

![view-product-catalog-api.json in the IDE](images/activity3b-04-view-product-catalog-api-json.png)
![Gateway detail page: Products and Sales targets both Ready](images/activity3b-05-gateway-targets-products-sales.png)

## Wiring it into the agent and testing

`activity3b_gen.sh` follows the same pattern, updating both the tool connection and the system
prompt's description of what the agent can now do:

![Running activity3b_gen.sh](images/activity3b-06-activity3b-gen-script-start.png)
![agent.py system prompt update reflecting the new capability](images/activity3b-07-agent-py-system-prompt-update.png)
![Redeploying via launchAgent.sh](images/activity3b-08-launchagent-redeploy-progress.png)
![Deployment complete](images/activity3b-09-deployment-complete.png)

Next: [Activity 3c - Deploy the Customer Reviews tool](03c-activity3c-deploy-the-customer-reviews-tool.md).

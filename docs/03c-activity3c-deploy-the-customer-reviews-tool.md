# Activity 3c: Deploy the Customer Reviews tool

Part of [Activity 3: Deploy the Sales, Products, and Reviews tools](03-activity3-sales-products-reviews-tools.md).
Follows [Activity 3b - Products](03b-activity3b-deploy-the-products-tool.md).

## Architecture after this activity

```mermaid
graph LR
    subgraph TD1["Trust Domain 1 - AnyCompany"]
        User["Authenticated user"] -->|Cognito JWT| Runtime["AgentCore Runtime<br/>Strands Agent"]
        Runtime -->|OAuth2 client credentials| GW2["AgentCore Gateway<br/>AnyCompany-Sales-Products-Reviews-Tool"]
        GW2 --> Sales["Sales API Gateway + Lambda"]
        GW2 --> Products["Products API Gateway + Lambda"]
        GW2 --> Reviews["Customer Reviews API Gateway + Lambda"]
        GW1["AgentCore Gateway<br/>AnyCompany-ToS-Tool"]
        Runtime -.-> GW1
    end
```

All three Activity 3 targets are now live behind the one gateway - this is the end state carried
into [Activity 4](04-activity4-add-the-inventory-mcp-tool.md).

## Adding the third target to the same gateway

![Add target: connector configuration for Customer Reviews](images/activity3c-01-add-target-connectors.png)
![Running activity3c_gen.sh](images/activity3c-02-activity3c-gen-script-terminal.png)
![Redeploy via launchAgent.sh](images/activity3c-03-launchagent-redeploy-script.png)

## Testing the complete Activity 3 toolset

With all three Activity 3 tools live, the agent can answer sales, catalog, and review questions in
the same conversation:

![Chat test: customer reviews query](images/activity3c-04-agent-customer-reviews-query.png)
![Chat test: detailed customer reviews response](images/activity3c-05-agent-customer-reviews-detailed.png)
![Chat test: best-rated products analysis](images/activity3c-06-agent-best-rated-products.png)

Next: [Activity 4 - Add the Inventory MCP Tool](04-activity4-add-the-inventory-mcp-tool.md).

# AWS Services Used

Official AWS Architecture Icons, used under [AWS's stated icon usage
guidelines](https://aws.amazon.com/architecture/icons/) to describe how each service is used in
this project. Not an official AWS asset or endorsement - see the
[disclaimer](../README.md#disclaimer).

| Service | Role in this project |
|---|---|
| **Amazon Bedrock AgentCore Runtime** | Hosts the Strands agent itself; validates the inbound Cognito JWT via a `customJWTAuthorizer` before any request reaches the agent code |
| **Amazon Bedrock AgentCore Gateway** | Exposes each backend (Lambda, API Gateway, external MCP server) as MCP tools behind a single inbound-auth boundary (IAM/SigV4 in Activity 2, Cognito OAuth2 from Activity 3 onward) |
| **Amazon Bedrock AgentCore Identity** | Outbound credential vault - mints short-lived OAuth2 access tokens for machine-to-machine calls (Activity 4's cross-trust-domain Inventory call) so no client secret lives in the agent's code |
| **Amazon Bedrock (foundation model)** | Runs the Strands `Agent` reasoning loop against `global.anthropic.claude-opus-4-6-v1` |
| **Amazon Cognito** | User pool + app clients backing both the agent's own inbound JWT auth and the OAuth2 client-credentials flow between the agent and each Cognito-fronted gateway |
| **Amazon Verified Permissions** | Cedar policy store evaluated per-request (`batch_is_authorized_with_token`) to decide which tool groups and individual tools a given authenticated caller may load (Activity 5) |
| **Amazon API Gateway** | Fronts the Sales, Products, Driver, and Truck Lambda-backed REST endpoints exposed as AgentCore Gateway targets |
| **AWS Lambda** | Backend logic for each tool (Terms of Service lookup, sales/inventory/product/review queries) |
| **Amazon S3** | Private schema bucket for the OpenAPI/JSON schemas consumed by AgentCore Gateway target configuration, accessed via presigned URLs |
| **AWS CodeBuild** | Builds and pushes the agent's container image during `agentcore launch` (the deploy scripts poll its build phases) |
| **Amazon ECR** | Stores the built agent runtime container image |
| **AWS CloudFormation** | Provisions the workshop's `bootstrap-stack`, `direct-targets-stack`, and `standalone-mcp-server-stack`, and is queried by the `activity*_gen.sh` scripts for stack outputs (gateway URLs, OAuth provider names) |
| **AWS IAM** | Execution role for the agent runtime (used for SigV4-signed calls to the Activity 2 gateway) and the Lambda functions' own roles |
| **Amazon CloudWatch** | Log delivery destination configured for the AgentCore Runtime (`USAGE_LOGS` delivery, seen in the console during Activity 1 setup) |

## Tech stack outside AWS

**Strands Agents SDK** (agent framework, tool orchestration) · **Model Context Protocol (MCP)**
(the tool-calling protocol between the agent and every gateway) · **Python** (agent + tooling) ·
**Bash** (deployment and activity-wiring scripts) · **Cedar** (the policy language behind Verified
Permissions)

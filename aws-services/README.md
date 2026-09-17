# AWS Services Used

Every service below is mapped to the specific role it plays in this architecture - not just
listed, but tied back to the activity that shows it running. For the full topology, see
[`../architecture/system-architecture.md`](../architecture/system-architecture.md); this page is
the service-level index into it.

## Services at a glance

| Icon | Service | Category | Role in this project | Where it's covered |
|:---:|---|---|---|---|
| <img src="icons/agentcore.png" width="48" alt="Amazon Bedrock AgentCore"> | **Amazon Bedrock AgentCore** (Runtime, Gateway, Identity) | Artificial Intelligence | Hosts the agent (Runtime), exposes every backend as an MCP tool behind one inbound-auth boundary per gateway (Gateway), and mints short-lived outbound credentials for cross-account calls (Identity). AWS doesn't yet publish separate icons for these three capabilities, so one icon stands for all three here, distinguished by label. | [Activities 1-5](../docs) |
| <img src="icons/bedrock.png" width="48" alt="Amazon Bedrock"> | **Amazon Bedrock** | Artificial Intelligence | Runs the foundation model (`global.anthropic.claude-opus-4-6-v1`) behind the Strands `Agent` reasoning loop. | [Activity 1](../docs/01-activity1-introduction-to-your-ai-agent.md) |
| <img src="icons/cognito.png" width="48" alt="Amazon Cognito"> | **Amazon Cognito** | Security, Identity & Compliance | User pool + app clients backing both the agent's own inbound JWT auth and the OAuth2 client-credentials flow between the agent and each Cognito-fronted gateway. | [Activity 1](../docs/01-activity1-introduction-to-your-ai-agent.md) · [Activity 3a](../docs/03a-activity3a-deploy-the-sales-tool.md) |
| <img src="icons/verified-permissions.png" width="48" alt="Amazon Verified Permissions"> | **Amazon Verified Permissions** | Security, Identity & Compliance | Cedar policy store evaluated per-request to decide which tool groups and individual tools a given authenticated caller may load. | [Activity 5](../docs/05-activity5-dynamic-tool-filtering-capability.md) |
| <img src="icons/api-gateway.png" width="48" alt="Amazon API Gateway"> | **Amazon API Gateway** | Networking & Content Delivery | Fronts the Sales, Products, Driver, Truck, and Inventory Lambda-backed REST endpoints exposed as AgentCore Gateway targets - Sales via a bare API key, Products (and the vendor's Inventory endpoint) behind a Lambda Authorizer validating an OAuth2 token. | [Activity 3a](../docs/03a-activity3a-deploy-the-sales-tool.md) · [3b](../docs/03b-activity3b-deploy-the-products-tool.md) · [4](../docs/04-activity4-add-the-inventory-mcp-tool.md) |
| <img src="icons/lambda.png" width="48" alt="AWS Lambda"> | **AWS Lambda** | Compute | Backend logic for each API Gateway-fronted tool - Terms of Service lookup, sales/inventory/product queries, and each target's Lambda Authorizer. | [Activity 2](../docs/02-activity2-add-the-terms-of-service-tool.md) |
| <img src="icons/dynamodb.png" width="48" alt="Amazon DynamoDB"> | **Amazon DynamoDB** | Database | Backs the Customer Reviews tool directly - no Lambda or API Gateway in front of it, reached through an AgentCore Gateway *native integration* target, authenticated with a plain IAM role. | [Activity 3c](../docs/03c-activity3c-deploy-the-customer-reviews-tool.md) |
| <img src="icons/s3.png" width="48" alt="Amazon S3"> | **Amazon S3** | Storage | Private schema bucket for the OpenAPI/JSON schemas consumed by AgentCore Gateway target configuration, accessed via presigned URLs. | [`code/schemas/README.md`](../code/schemas/README.md) |
| <img src="icons/codebuild.png" width="48" alt="AWS CodeBuild"> | **AWS CodeBuild** | Developer Tools | Builds and pushes the agent's container image during `agentcore launch`. | [`architecture/system-architecture.md`](../architecture/system-architecture.md#the-deployment-pipeline-used-identically-by-every-activity) |
| <img src="icons/ecr.png" width="48" alt="Amazon ECR"> | **Amazon ECR** | Containers | Stores the built agent runtime container image. | Same as above |
| <img src="icons/cloudformation.png" width="48" alt="AWS CloudFormation"> | **AWS CloudFormation** | Management & Governance | Provisions the workshop's `bootstrap-stack`, `direct-targets-stack`, and `standalone-mcp-server-stack`; queried by every `activity*_gen.sh` script for stack outputs. | [`code/README.md`](../code/README.md) |
| <img src="icons/iam.png" width="48" alt="AWS IAM"> | **AWS IAM** | Security, Identity & Compliance | Execution role for the agent runtime (SigV4-signed calls to the Activity 2 gateway) and each Lambda's own role. | [Activity 2](../docs/02-activity2-add-the-terms-of-service-tool.md) |
| <img src="icons/cloudwatch.png" width="48" alt="Amazon CloudWatch"> | **Amazon CloudWatch** | Management & Governance | Log delivery destination configured for the AgentCore Runtime (`USAGE_LOGS` delivery). | [Activity 1](../docs/01-activity1-introduction-to-your-ai-agent.md) |

## How the services connect - Activity 2 (IAM/SigV4 gateway)

<table>
<tr>
<td align="center"><img src="icons/cognito.png" width="40"><br><sub>Cognito<br/>(inbound JWT)</sub></td>
<td align="center">→</td>
<td align="center"><img src="icons/agentcore.png" width="40"><br><sub>AgentCore<br/>Runtime</sub></td>
<td align="center">→</td>
<td align="center"><img src="icons/iam.png" width="40"><br><sub>IAM role<br/>(SigV4)</sub></td>
<td align="center">→</td>
<td align="center"><img src="icons/agentcore.png" width="40"><br><sub>AgentCore<br/>Gateway</sub></td>
<td align="center">→</td>
<td align="center"><img src="icons/lambda.png" width="40"><br><sub>ToS<br/>Lambda</sub></td>
</tr>
</table>

The simplest chain in the project: the agent's own execution role is the only credential needed,
because everything stays inside one AWS account.

## How the services connect - Activities 3a/3b/3c (one gateway, three different backends)

One gateway, one inbound JWT (2LO) auth mode from the Runtime - but three completely different
outbound chains once inside it, matched to what each backend actually needs:

<table>
<tr>
<td align="center"><img src="icons/agentcore.png" width="40"><br><sub>AgentCore<br/>Gateway</sub></td>
<td align="center">→ API Key →</td>
<td align="center"><img src="icons/api-gateway.png" width="40"><br><sub>Sales<br/>API Gateway</sub></td>
<td align="center">→</td>
<td align="center"><img src="icons/lambda.png" width="40"><br><sub>Sales<br/>Lambda</sub></td>
</tr>
<tr>
<td align="center"><img src="icons/agentcore.png" width="40"><br><sub>AgentCore<br/>Gateway</sub></td>
<td align="center">→ OAuth2 (2LO) →</td>
<td align="center"><img src="icons/api-gateway.png" width="40"><br><sub>Products API Gateway<br/>(Lambda Authorizer)</sub></td>
<td align="center">→</td>
<td align="center"><img src="icons/lambda.png" width="40"><br><sub>Products<br/>Lambda</sub></td>
</tr>
<tr>
<td align="center"><img src="icons/agentcore.png" width="40"><br><sub>AgentCore<br/>Gateway</sub></td>
<td align="center">→ IAM role →</td>
<td align="center" colspan="3"><img src="icons/dynamodb.png" width="40"><br><sub>Customer Reviews - Amazon DynamoDB<br/>(native integration, no Lambda/API Gateway)</sub></td>
</tr>
</table>

See [`architecture/system-architecture.md`](../architecture/system-architecture.md) for why one
gateway with heterogeneous targets still beats standing up a new gateway per tool - the inbound
trust boundary (gateway ↔ Runtime) and each target's outbound auth are independent axes.

## How the services connect - Activity 4 (cross-trust-domain, two gateways)

<table>
<tr>
<td align="center"><img src="icons/agentcore.png" width="40"><br><sub>AgentCore<br/>Identity</sub></td>
<td align="center">→ M2M token →</td>
<td align="center"><img src="icons/agentcore.png" width="40"><br><sub>AgentCore Gateway<br/>AnyCompany-Inventory-Tool<br/>(AnyCompany-owned)</sub></td>
<td align="center">→ 2nd hop, OAuth2 (2LO) →</td>
<td align="center"><img src="icons/agentcore.png" width="40"><br><sub>Vendor's own<br/>AgentCore Gateway</sub></td>
<td align="center">→</td>
<td align="center"><img src="icons/api-gateway.png" width="40"><br><sub>Inventory API Gateway<br/>(Lambda Authorizer)</sub></td>
<td align="center">→</td>
<td align="center"><img src="icons/lambda.png" width="40"><br><sub>Inventory<br/>Lambda</sub></td>
</tr>
</table>

The only chain in the project that leaves AnyCompany's own AWS account - and it does so through a
gateway AnyCompany itself owns, not directly into the vendor's domain. The vendor runs their own
independent Cognito pool and AgentCore Identity behind their gateway; AnyCompany's AgentCore
Identity only ever mints the credential for the *first* hop, into AnyCompany's own gateway. See
[Activity 4](../docs/04-activity4-add-the-inventory-mcp-tool.md) for the full explanation of why
two gateways are involved when the agent's own code only ever calls one.

## How the services connect - Activity 5 (authorization gate)

<table>
<tr>
<td align="center"><img src="icons/agentcore.png" width="40"><br><sub>AgentCore<br/>Runtime</sub></td>
<td align="center">→</td>
<td align="center"><img src="icons/verified-permissions.png" width="40"><br><sub>Verified<br/>Permissions</sub></td>
<td align="center">→</td>
<td align="center">✅ / ⛔<br><sub>allowed tool groups<br/>only</sub></td>
</tr>
</table>

This gate sits in front of every gateway above - it doesn't replace any of the other chains, it
decides which of them even get opened for a given authenticated caller.

## Cross-cutting

<table>
<tr>
<td align="center"><img src="icons/cloudformation.png" width="40"><br><sub>CloudFormation</sub></td>
<td align="center"><img src="icons/codebuild.png" width="40"><br><sub>CodeBuild</sub></td>
<td align="center"><img src="icons/ecr.png" width="40"><br><sub>ECR</sub></td>
<td align="center"><img src="icons/cloudwatch.png" width="40"><br><sub>CloudWatch</sub></td>
<td align="center"><img src="icons/s3.png" width="40"><br><sub>S3</sub></td>
</tr>
</table>

These aren't specific to any one activity: CloudFormation provisions every stack, CodeBuild/ECR
build and store every deployed image, CloudWatch receives every runtime's logs, and S3 holds the
schemas every gateway target is configured from. See
[`architecture/system-architecture.md#the-deployment-pipeline-used-identically-by-every-activity`](../architecture/system-architecture.md#the-deployment-pipeline-used-identically-by-every-activity).

---

**Icon credit:** service icons are AWS's official [Architecture
Icons](https://aws.amazon.com/architecture/icons/), used under AWS's icon usage guidelines to
describe how AWS services are used in this project. This repository is not affiliated with or
endorsed by Amazon Web Services.

# Code - Real Workshop Source

Unlike a reconstruction from screenshots, everything in this folder is the **actual source
distributed by the workshop's Cloud IDE**, copied out of the lab environment as-is. Nothing here
is rebuilt from what a screenshot happened to show.

## Licensing - what's covered and what isn't

Only [`agent_core.py`](agent_core.py) carries an explicit license header: AWS's own
**MIT-No-Attribution (MIT-0)** notice, `Copyright 2025 Amazon`. That's what makes redistributing
*that specific file* here unambiguous, and the header is kept intact.

None of the other files - the two transport modules, the five `activity*_gen.sh` generator
scripts, the two deployment scripts, or the six schema files in [`schemas/`](schemas) - carry an
explicit license header of their own. They're workshop lab material handed to every participant
through the Cloud IDE to complete the exercises, not independently-published open-source code, so
I'm not asserting a license for them the way the MIT-0 header lets me for `agent_core.py`. They're
included here as evidence of the hands-on work actually done (same spirit as embedding a
screenshot), not as a redistributable library.

## What `agent_core.py` actually is

This is the **starting template** handed out at the beginning of the lab, not the fully wired
agent from the end of it. Every activity's insertion point is present but empty:

```python
'''START OF ACTIVITY 2'''

'''END OF ACTIVITY 2'''
```

The `activity*_gen.sh` scripts are what fill those blocks in, by fetching a value from
CloudFormation (a gateway URL, an OAuth2 credential provider name) and `awk`-splicing a snippet
between the matching markers - see [`activity2_gen.sh`](activity2_gen.sh) for the simplest example
and [`activity4_gen.sh`](activity4_gen.sh) for the OAuth2-credential-provider variant. The actual
*filled-in* `agent.py` at each stage was never re-exported from the IDE - it only exists as what's
visible in the terminal output and editor screenshots embedded in [`../docs/`](../docs) (e.g. the
gen script's own success/diff output, or an editor screenshot after it ran). Treat this file as the
scaffold the whole lab builds on top of, and the docs as the record of what actually got built into
it at each step.

One more marker pair, `START OF ACTIVITY 6` / `END OF ACTIVITY 6`, is also present and empty. No
Activity 6 shows up anywhere in the schemas, gen scripts, or screenshots for this lab run - it's
left in the template here unexplained rather than invented.

## Files

| File | What it is |
|---|---|
| [`agent_core.py`](agent_core.py) | The Strands/BedrockAgentCoreApp agent template (MIT-0, Amazon) - system prompt, tool-group registry, per-request AVP-gated tool loading, the three transport-builder functions (SigV4, OAuth2 client-credentials, AgentCore Identity M2M) |
| [`streamable_http_sigv4.py`](streamable_http_sigv4.py) | MCP StreamableHTTP transport extended with AWS SigV4 request signing, for gateways with IAM inbound auth |
| [`streamable_http_oauth2.py`](streamable_http_oauth2.py) | MCP StreamableHTTP transport extended with OAuth2 client-credentials auth, for gateways with Cognito inbound auth |
| [`activity2_gen.sh`](activity2_gen.sh) | Wires the Activity 2 Terms-of-Service gateway URL into `agent.py` |
| [`activity3a_gen.sh`](activity3a_gen.sh), [`activity3b_gen.sh`](activity3b_gen.sh), [`activity3c_gen.sh`](activity3c_gen.sh) | Wire the Sales, Products, and Customer Reviews gateway targets into `agent.py` |
| [`activity4_gen.sh`](activity4_gen.sh) | Wires the external Inventory MCP server's gateway URL and AgentCore Identity OAuth2 provider name into `agent.py` |
| [`launchagent.sh`](launchagent.sh) | Thin wrapper that calls the deploy script |
| [`deploy-agentcore-runtime.sh`](deploy-agentcore-runtime.sh) | Full deploy workflow: packages the agent, runs `agentcore configure`/`launch` (the Python `bedrock-agentcore-starter-toolkit`) with a Cognito JWT customJWTAuthorizer and header allowlist |
| [`requirements.txt`](requirements.txt) | Pinned dependencies (`bedrock-agentcore`, `strands-agents`, `bedrock-agentcore-starter-toolkit==0.3.9`, `mcp>=1.9.0,<2.0.0`, boto3/botocore/requests) |
| [`schemas/`](schemas) | OpenAPI 3.0.3 specs for the five API Gateway-backed tools (sales, inventory, products, driver, truck) and a JSON Schema for the Terms-of-Service Lambda tool, plus the workshop's own schema README |

## What's deliberately not here

No CloudFormation/SAM templates for the `bootstrap-stack`, `direct-targets-stack`, or
`standalone-mcp-server-stack` seen in the console screenshots - those provision the gateways,
Cognito pools, and Lambda functions behind the scenes and were never exposed as files in the IDE,
only as deployed resources. No Cedar policy documents either (the six policies created in Activity
5 - Terms of Service, Sales, Reviews, Admin, Inventory) - two are shown as code in the workshop's
own guide screenshots (embedded and described in
[`docs/05-activity5-dynamic-tool-filtering-capability.md`](../docs/05-activity5-dynamic-tool-filtering-capability.md))
but the full set was authored directly in the Verified Permissions console, not as files.

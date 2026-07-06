---
title: MCP Servers
description: Extend the platform's tool inventory with self-hosted or remote MCP servers.
---

# MCP Servers

The Model Context Protocol (MCP) is an open specification for exposing tools to LLMs. OutcomeOps AI Assist can consume MCP servers so users can invoke tools from within chat --- SonarQube analysis, AWS CLI calls, custom internal APIs --- without waiting for OutcomeOps to ship a native integration.

Every MCP server appears in the chat composer's **MCP picker**. Users check the ones they want available for the current turn, and the LLM gets those servers' tools in its tool inventory.

## Two kinds of MCP servers

**OutcomeOps-hosted catalog MCPs** ship with the platform as opt-in Fargate containers. Set the tfvar to `true`, run `make pre-deploy` to build the image, and the platform provisions a Fargate service running that MCP. The current catalog:

- **SonarQube** --- code-quality analysis, issue triage, quality gate status.
- **Snyk** --- dependency vulnerability + license risk analysis (optional).

Add more by contributing to the catalog Terraform module.

**Remote / user-added MCPs** are external endpoints you point the platform at. Add them from the Admin → MCP Servers page in the UI. The user provides a URL and (optionally) a bearer token or API key; the platform stores credentials as a KMS-encrypted DynamoDB item and calls the endpoint from the chat backend at tool invocation time.

Examples that customers have added:

- **AWS MCP Server** --- Amazon's official MCP for AWS CLI + skill access.
- **Custom internal APIs** --- point at a company-hosted MCP wrapping your ticketing system, deployment tool, or ops runbook.
- **Vendor MCPs** --- any third-party MCP server that speaks the standard protocol.

## Global vs. workspace-scoped

**Global MCP servers** appear in every user's chat picker. Use these for tools the whole company benefits from --- the AWS MCP, SonarQube.

**Workspace-scoped MCP servers** only appear when a member of that workspace is chatting with it selected. Use these for team-specific tools --- the platform team's incident-response MCP, the data team's warehouse MCP.

An org admin decides which MCPs are global. Workspace admins can attach any global MCP to their workspace + add workspace-scoped ones.

## Adding a remote MCP from the UI

1. Sign in as an **org admin** (for global MCPs) or a **workspace admin** (for workspace-scoped).
2. Navigate to **Admin → MCP Servers** (org-level) or **Workspace Settings → MCP Servers** (workspace-level).
3. Click **Add MCP Server**.
4. Fill in:
    - **Name** --- displayed in the chat picker.
    - **URL** --- the MCP endpoint (e.g., `https://aws-mcp.us-east-1.api.aws/mcp`).
    - **Auth type** --- None / Bearer / API Key.
    - **Credential** (if applicable) --- pasted here, encrypted before landing in DynamoDB.
    - **Global** --- check to make it available across every workspace.
5. Save. The platform fetches the MCP's `tools/list` and stores the tool schema in DynamoDB.
6. The MCP appears in the picker with a tool count.

## How tool selection works

When a user selects an MCP in the chat picker and sends a message:

1. The chat backend loads the MCP's cached tool schema from DynamoDB.
2. It prefixes each tool name with the MCP's ID (e.g., `catalog-sonarqube___search_my_sonarqube_projects`) to avoid name collisions when multiple MCPs are attached.
3. It builds the tool inventory and sends it to Bedrock along with the user's message.
4. If the LLM chooses to invoke a tool, the backend un-prefixes the name, dispatches the JSON-RPC call to the MCP's endpoint, and threads the response back to the LLM for the next synthesis step.

Tool calls are logged in the audit trail with the tool name, arguments, and result --- so SecOps can see exactly which MCP tools got invoked, by whom, in response to what prompt.

## RAG vs. MCP tools: the priority behavior

When both apply to a user's question --- retrieval finds documents AND the attached MCP has a tool that could answer --- the current model tends to **prefer the retrieved documents**. This isn't a bug; the system prompt biases toward "answer definitively based on knowledge base context" because most user questions in developer contexts benefit from citations over tool calls.

If you want the LLM to definitely invoke the tool, be explicit: **"use the AWS MCP and list all regions"** instead of just **"list all AWS regions."** The explicit phrasing overrides the default RAG-first bias.

This is a known model behavior; if it becomes a common friction point for your users, we can update the system prompt to prefer tool calls when live-data-shaped questions come in. See the FAQ for the current recommendation.

## Common problems

| Symptom | Cause | Fix |
| --- | --- | --- |
| MCP shows in picker with "0 tools" | Discovery to the endpoint failed --- unreachable, auth wrong, or the endpoint doesn't speak MCP protocol. | Click **Refresh tools** on the MCP row. If still 0, check the endpoint from a machine that can reach it: `curl -sX POST <url> -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'`. |
| Tool call fails with 401 / 403 | Bearer token expired or was wrong. | Edit the MCP, paste a fresh credential, save. |
| Tool inventory too large --- LLM ignores tools | The MCP exposes 50+ tools and the LLM gets overwhelmed. | Attach fewer MCPs per turn, or ask the MCP owner to split the toolset into narrower servers. |
| Global MCP not appearing for one user | User's session cached an older MCP list. | Reload the browser. |

## Removing an MCP server

Admin → MCP Servers → find the entry → **Delete**. The DynamoDB rows (METADATA + TOOLS) are removed. Chats that had this MCP selected in prior conversations retain the historical tool calls in their audit trail but can't invoke the tool again.

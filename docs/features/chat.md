---
title: Chat
description: How the OutcomeOps chat surface works -- workspace-scoped retrieval, source citations, streaming responses.
---

# Chat

Chat is the primary user surface. It's a streaming conversation with an LLM, scoped to whichever workspaces the user selects, grounded in citations drawn from the workspaces' ingested content.

## The composer bar

At the bottom of the chat page, three pickers control what the current turn will use:

- **Workspaces** --- one or more workspaces the user is a member of (plus any [global workspaces](workspaces.md#global-workspaces)). Retrieval happens across the union of their content.
- **MCP servers** --- optional. Any [MCP server](mcp-servers.md) the user selects becomes part of the LLM's tool inventory for this turn.
- **Prompt / voice** --- optional. Pre-canned instructions that steer the LLM's tone or shape of the answer (e.g., "explain like I'm a new engineer," "return only code").

Type a message, press Enter, watch the response stream in token-by-token.

## Sources panel

Every response the LLM grounds in retrieved content shows citations in a **Sources** panel to the right of the message. Click a source to jump to the exact file + line range the platform pulled from. This is the audit story: every claim in the response is either grounded in code (or docs, or tickets) you can inspect, or the LLM is answering from general knowledge with no citation.

Sources are typed:

- **Code** --- a file + line range in a repo. Common for code-map questions.
- **Docs** --- a Confluence page, README, or ADR.
- **Tickets** --- a Jira issue or GitHub Issue, cited by issue key.
- **Schema** --- a database table or column.

Different types cluster in the panel so a code-heavy answer's code sources appear together, docs together, etc.

## Retrieval, in one paragraph

For every user turn, the platform runs a **dual-channel retrieval** query in parallel: a **code-map lane** (looks for symbol-graph hits --- functions, classes, definitions, callers) and a **docs lane** (looks for prose --- Confluence pages, READMEs, Jira descriptions). Each lane gets half the candidate budget. Both lanes run every turn regardless of what the question looks like, so the LLM synthesis step always sees both structural truth (code) and authored narrative (docs). When they conflict, code wins.

If reranking is enabled, results from both lanes are reranked with a Bedrock rerank model before landing in the LLM's context window. Reranking is optional (some AWS regions don't offer Cohere Rerank), and even without it the dual-channel design gets you most of the way.

## Conversation memory

Long conversations get summarized. When a chat runs past a threshold (default ~50 turns), the platform writes a running summary of earlier turns to the conversation memory table. Subsequent turns include the summary in the LLM's context so the model doesn't lose the plot in hour-long sessions. The summary is workspace-scoped and never leaks across workspaces.

You can see and edit the summary from the chat header --- click the settings icon → **Show conversation summary**.

## Prompt / voice picker

Workspace admins can pre-can prompts that steer the LLM's output. Examples:

- **"Explain like a new engineer"** --- adds instructions to define jargon inline and skip advanced context.
- **"Return only code"** --- suppresses prose and returns just the code snippet.
- **"Compliance-flavored"** --- prompts the model to reference security standards + policies more prominently.

Prompts are managed per workspace under Workspace Settings → Prompts. Users select from the picker before sending a message.

## Guardrails

- **Refusal patterns.** The platform refuses to help with certain classes of requests (safety-flagged prompts) and returns a brief refusal.
- **PII redaction.** If PII redaction is enabled at the org or workspace level, personally identifiable information in retrieved content is redacted before it reaches the LLM. See [PII Redaction](../administration/pii-redaction.md).
- **Read-only tool inventory.** MCP servers you attach to a chat can only invoke tools whose schema the platform recognizes. Tool inputs are validated before dispatch.

## Model selection

Two Bedrock models fire per turn:

- **Advanced model** (default Claude Sonnet) --- runs the synthesis step that produces the streamed answer.
- **Basic model** (default Claude Haiku) --- runs auxiliary steps like retrieval query rewriting and conversation summarization.

Both are swappable via tfvars for compliance customers. Regions without Claude get OpenAI on Bedrock (GPT-5.5) via a shared model client abstraction. See [Deploy](../getting-started/deploy.md) for the model tfvars.

## What chat is not

- **Not a code editor.** The platform doesn't modify files in your repos. Code output happens through [code generation](code-generation.md), which opens PRs you review.
- **Not a general-purpose assistant.** Retrieval + tool inventory are scoped to the workspaces and MCP servers you selected. It won't answer questions about content you haven't given it access to.
- **Not stateless.** Conversations persist per user + workspace. Reload the page and the conversation is still there.

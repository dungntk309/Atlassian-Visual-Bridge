# Atlassian Visual Bridge

A lightweight, serverless bridge that lets AI agents retrieve **authenticated Jira and Confluence attachments** through GitHub Actions when a connector can read metadata but cannot directly provide the underlying file bytes.

> Built as a public portfolio / reference implementation. Use a private repository for confidential Atlassian data.

## The problem

AI connectors can often read Jira/Confluence text and attachment metadata, but visual analysis still requires the **real binary file**.

That creates a practical integration gap:

```text
Connector can read metadata
        ↓
Agent knows an attachment exists
        ↓
But vision / file analysis cannot access the authenticated binary
```

Atlassian Visual Bridge fills that gap without requiring a VPS, always-on proxy, or local MCP server.

## Architecture

```text
AI Agent / Automation
        │
        │ attachment/content ID
        ▼
Request JSON in GitHub
        │
        │ push / manual dispatch
        ▼
GitHub Actions
        │
        ├─ validate request
        ├─ load credentials from Secrets
        ├─ call Jira / Confluence REST API
        ├─ resolve metadata + filename
        └─ download authenticated binary
        │
        ▼
Short-lived GitHub Actions Artifact
        │
        ▼
AI vision / file analysis
```

The bridge deliberately separates **structured metadata retrieval** from **binary transport**. Existing Jira/Confluence connectors remain responsible for issues, pages, comments, and metadata; this project only handles the fallback path for authenticated files.

## What it demonstrates

- GitHub Actions as an **on-demand serverless integration layer**
- event-driven workflows triggered by request files
- Jira and Confluence REST API integration
- GitHub Actions Secrets for credential isolation
- Bash automation with `curl`, `jq`, and strict error handling
- input validation before external API calls
- retry handling for transient HTTP failures
- filename sanitization before writing downloaded files
- single and batch attachment processing
- short-lived artifact delivery for downstream automation

### Tech stack

| Area | Technology |
|---|---|
| Automation | GitHub Actions |
| Integration | Jira REST API, Confluence REST API |
| Runtime | Ubuntu GitHub-hosted runner |
| Scripting | Bash |
| HTTP | `curl` |
| JSON | `jq` |
| Credentials | GitHub Actions Secrets |
| Binary delivery | GitHub Actions Artifacts |

## Workflows

| Workflow | Trigger | Purpose | Output |
|---|---|---|---|
| Jira Attachment Bridge | `requests/*.json` or manual dispatch | Download one Jira attachment | `jira-attachment-<id>` |
| Confluence Attachment Bridge | `confluence-requests/*.json` | Download one Confluence attachment | `confluence-attachment-<id>` |
| Confluence Batch Bridge | `confluence-batch-requests/*.json` | Download multiple attachments from one page | `confluence-batch-<contentId>` |

### Example: Jira attachment

```json
{
  "issueKey": "DEMO-123",
  "attachmentId": "12345"
}
```

Push the request under:

```text
requests/DEMO-123-12345.json
```

The workflow then:

```text
validate attachmentId
→ fetch Jira attachment metadata
→ resolve filename / MIME type
→ download the binary using authenticated API access
→ generate manifest.json
→ upload the result as a temporary artifact
```

### Example: Confluence batch

```json
{
  "contentId": "123456789",
  "attachmentIds": [
    "att111111111",
    "att222222222",
    "att333333333"
  ]
}
```

This is useful for PRDs or technical pages containing multiple screenshots, diagrams, wireframes, or user flows.

## Engineering decisions

### Why GitHub Actions?

The bridge is designed for **on-demand** file retrieval rather than a continuously running service. GitHub Actions provides ephemeral compute, secret management, logs, retries, and artifact storage without introducing another hosted backend.

### Why request files?

A request JSON acts as a simple job envelope:

- easy for automation or an AI agent to generate
- versioned and auditable through Git
- easy to validate with `jq`
- naturally compatible with path-based GitHub Actions triggers

### Why artifacts?

Artifacts provide a temporary handoff point between the authenticated download step and the downstream consumer. The current workflows retain artifacts for only **1 day** to keep the bridge temporary by design.

## Validation & failure handling

The workflows fail early instead of silently producing incomplete output.

Examples include:

- Jira `attachmentId` must be numeric
- Confluence attachment IDs must match the expected `att...` format
- batch requests must contain at least one attachment
- required GitHub Secrets are checked before API calls
- `curl --fail` turns HTTP errors into workflow failures
- downloads use retries for transient failures
- filenames are reduced to their basename before writing to disk

This makes failures visible in GitHub Actions logs instead of allowing missing visual evidence to look like a successful review.

## Security model

```text
Repository request
      │ identifiers only
      ▼
GitHub Actions runner
      │
      ├─ JIRA_BASE_URL
      ├─ JIRA_EMAIL
      └─ JIRA_API_TOKEN
           ↑
     GitHub Actions Secrets
```

Key rules:

- credentials stay in GitHub Actions Secrets
- request files contain identifiers, not credentials
- workflows use minimal repository permissions
- artifacts expire after 1 day
- no authentication bypass is performed
- the Atlassian account still needs permission to access the requested content
- never commit production tokens, signed URLs, or confidential attachments to a public fork

See [SECURITY.md](SECURITY.md) for details.

## Quick setup

Configure these repository secrets:

```text
JIRA_BASE_URL
JIRA_EMAIL
JIRA_API_TOKEN
```

Repository layout:

```text
.github/workflows/
├── jira-attachment-bridge.yml
├── confluence-attachment-bridge.yml
└── confluence-batch-bridge.yml

requests/
confluence-requests/
confluence-batch-requests/
examples/
```

Then create a request JSON and push it to the matching request directory.

## Scope

Implemented:

- Jira single attachment bridge
- Confluence single attachment bridge
- Confluence batch attachment bridge
- request validation
- authenticated binary download
- metadata / manifest generation
- temporary artifact delivery

Out of scope for now:

- generic arbitrary-URL downloader
- long-running proxy service
- local MCP server
- permission bypass or credential delegation

## Design principle

This project is a **binary transport fallback**, not a replacement for Jira or Confluence connectors.

Use connectors for structured text and metadata. Use this bridge only when an AI agent or automation needs the actual authenticated file bytes for vision or file analysis.

# Atlassian Visual Bridge

A lightweight, serverless bridge that lets AI agents retrieve **authenticated Jira and Confluence attachments** through GitHub Actions when a connector can read metadata but cannot directly provide the underlying file bytes.

> Built as a public portfolio / reference implementation. Use a private repository for confidential Atlassian data.

## Why

AI connectors can often read Jira/Confluence text and attachment metadata, but visual analysis still requires the **real binary file**.

```text
AI Agent
   ↓
Jira / Confluence metadata
   ↓
Request JSON pushed to GitHub
   ↓
GitHub Actions
   ↓
Authenticated Atlassian API download
   ↓
Short-lived GitHub Artifact
   ↓
AI vision / file analysis
```

No VPS, always-on proxy, or local MCP server is required.

## What it demonstrates

- GitHub Actions as an on-demand serverless integration layer
- Jira & Confluence REST API integration
- GitHub Actions Secrets for credential handling
- Bash + `curl` + `jq`
- request validation and fail-fast error handling
- single and batch attachment processing
- short-lived artifact delivery

## Workflows

| Workflow | Trigger | Output |
|---|---|---|
| Jira Attachment Bridge | `requests/*.json` or manual dispatch | `jira-attachment-<id>` |
| Confluence Attachment Bridge | `confluence-requests/*.json` | `confluence-attachment-<id>` |
| Confluence Batch Bridge | `confluence-batch-requests/*.json` | `confluence-batch-<contentId>` |

## Quick setup

Add these GitHub Actions Secrets:

```text
JIRA_BASE_URL
JIRA_EMAIL
JIRA_API_TOKEN
```

Example Jira request:

```json
{
  "issueKey": "DEMO-123",
  "attachmentId": "12345"
}
```

Save it under:

```text
requests/DEMO-123-12345.json
```

Then push the commit. GitHub Actions downloads the attachment and uploads it as a temporary artifact.

## Repository structure

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

## Security

- credentials stay in GitHub Actions Secrets
- request files contain identifiers, not credentials
- artifacts expire after 1 day
- workflows use minimal repository permissions
- never commit production tokens, signed URLs, or confidential attachments

See [SECURITY.md](SECURITY.md) for details.

## Scope

Implemented:

- Jira single attachment bridge
- Confluence single attachment bridge
- Confluence batch attachment bridge

This project is a **binary transport fallback**, not a replacement for Jira or Confluence connectors. It only transports files the configured Atlassian account is already authorized to access.

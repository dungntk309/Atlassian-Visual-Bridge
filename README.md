# Atlassian Visual Bridge

A lightweight, serverless binary bridge that helps AI agents retrieve **real Jira and Confluence attachments** through GitHub Actions when a connector can read metadata but cannot directly deliver the file bytes to vision.

> AI can understand images. The hard part is sometimes getting the authenticated binary file from Atlassian into the agent. This project handles that transport layer.

## Why this exists

An AI connector may be able to read Jira issues or Confluence pages and expose:

- issue/page text
- attachment metadata
- filename and MIME type
- attachment ID
- download URL

but still not provide the actual binary image/file to the model.

Atlassian Visual Bridge turns that into:

```text
AI Agent
  ↓
reads Jira / Confluence metadata
  ↓
creates a small request JSON in GitHub
  ↓
GitHub Actions runs with Atlassian credentials stored in Secrets
  ↓
downloads the real attachment
  ↓
uploads a short-lived GitHub Actions Artifact
  ↓
AI Agent downloads the Artifact
  ↓
Vision reads screenshot / diagram / wireframe / userflow
```

No local MCP server, VPS, or always-on proxy is required.

---

## Features

### Jira Attachment Bridge

Workflow:

```text
.github/workflows/jira-attachment-bridge.yml
```

Supports:

- automatic trigger from `requests/*.json`
- manual `workflow_dispatch`
- Jira attachment metadata lookup
- authenticated binary download
- `manifest.json` + original attachment
- artifact name: `jira-attachment-<attachmentId>`
- artifact retention: 1 day
- retry on transient download failures

### Confluence Attachment Bridge

Workflow:

```text
.github/workflows/confluence-attachment-bridge.yml
```

Supports:

- trigger from `confluence-requests/*.json`
- Confluence API v2 attachment metadata lookup
- authenticated binary download
- artifact name: `confluence-attachment-<attachmentId>`
- artifact retention: 1 day

For embedded Confluence images, resolve the exact image first:

```text
media UUID
→ attachment.fileId
→ attachment ID
→ bridge
→ binary image
```

Do **not** guess from filename, creation time, or attachment order.

### Confluence Batch Attachment Bridge

Workflow:

```text
.github/workflows/confluence-batch-bridge.yml
```

Useful for PRDs/pages containing many screenshots, userflows, wireframes, or diagrams.

Supports:

- multiple attachment IDs in one request
- validation of all IDs
- sequential download
- aggregated manifest
- one artifact: `confluence-batch-<contentId>`

---

## Repository structure

```text
Atlassian-Visual-Bridge/
├── .github/
│   └── workflows/
│       ├── jira-attachment-bridge.yml
│       ├── confluence-attachment-bridge.yml
│       └── confluence-batch-bridge.yml
├── requests/
├── confluence-requests/
├── confluence-batch-requests/
├── examples/
├── SECURITY.md
└── README.md
```

---

## Setup

### 1. Fork or copy this repository

For internal/company Jira or Confluence data, use a **private fork/repository**.

This public repository should be treated as a reusable template, not as a place to commit private ticket IDs or request payloads from production systems.

### 2. Create an Atlassian API token

Use an Atlassian account that already has permission to read the Jira/Confluence content you need.

You need:

```text
Atlassian Base URL
Atlassian Email
Atlassian API Token
```

### 3. Configure GitHub Actions Secrets

Go to:

```text
Repository
→ Settings
→ Secrets and variables
→ Actions
→ New repository secret
```

Create:

| Secret | Example / meaning |
|---|---|
| `JIRA_BASE_URL` | `https://your-domain.atlassian.net` |
| `JIRA_EMAIL` | Atlassian account email |
| `JIRA_API_TOKEN` | Atlassian API token |

Never commit these values to Git.

---

## Usage

### Jira — single attachment

Create a unique JSON file under `requests/`:

```json
{
  "issueKey": "DEMO-123",
  "attachmentId": "12345"
}
```

Example filename:

```text
requests/DEMO-123-12345-20260915T103000.json
```

Push the commit. GitHub Actions will run `Jira Attachment Bridge` and create:

```text
jira-attachment-12345
```

The artifact contains the original file plus metadata/manifest files.

You can also run the Jira workflow manually with:

- `attachment_id`
- `issue_key` (optional)

### Confluence — single attachment

Create:

```json
{
  "contentId": "123456789",
  "attachmentId": "att987654321"
}
```

under:

```text
confluence-requests/
```

Push the commit. Expected artifact:

```text
confluence-attachment-att987654321
```

### Confluence — batch attachments

Create:

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

under:

```text
confluence-batch-requests/
```

Expected artifact:

```text
confluence-batch-123456789
```

---

## Suggested AI-agent workflow

```text
1. Read Jira/Confluence text using the normal connector.
2. Inventory relevant visual evidence.
3. Resolve exact attachment IDs.
4. Create a bridge request JSON.
5. Push it to the repository.
6. Wait for the GitHub Actions run to complete.
7. Fetch the workflow artifact.
8. Extract the real file.
9. Open it with vision.
10. Compare visual evidence with requirement text.
```

A file should only be marked as visually reviewed after the actual binary has been opened by vision. Filename, metadata, media UUID, or download URL alone are not a visual review.

---

## Security model

- Credentials live only in GitHub Actions Secrets.
- Request files contain identifiers, not credentials.
- Workflows do not intentionally print tokens.
- Artifacts expire after 1 day.
- Use least-privilege Atlassian accounts/tokens.
- Prefer a private fork for internal data.
- Do not commit signed download URLs.
- Do not commit production screenshots or confidential attachments.

See [SECURITY.md](SECURITY.md).

---

## Troubleshooting

### Workflow does not start

Check that the request file is committed to the correct path:

```text
requests/*.json
confluence-requests/*.json
confluence-batch-requests/*.json
```

### Missing secret

Verify:

```text
JIRA_BASE_URL
JIRA_EMAIL
JIRA_API_TOKEN
```

### 401 / 403 from Atlassian

Check:

- API token validity
- email/token pairing
- Jira/Confluence permissions of the Atlassian account

### Confluence image mismatch

Do not pick a visually similar filename. Resolve the Confluence media UUID against attachment metadata and require:

```text
attachment.fileId == media UUID
```

### GitHub Action fails

Inspect:

```text
Actions
→ Workflow Run
→ Job
→ Step
→ Logs
```

Fix the transport failure instead of silently skipping the visual evidence.

---

## Current scope

Implemented now:

- Jira single attachment bridge
- Confluence single attachment bridge
- Confluence batch attachment bridge

Not implemented in this repository yet:

- Figma export bridge
- generic arbitrary-URL downloader
- local MCP server

The bridge does not bypass Atlassian permissions, authentication, API limits, or product safeguards. It only transports files the configured account is already authorized to access.

---

## Design principle

This project is a **binary transport fallback**, not a replacement for Jira/Confluence connectors.

Use the connector for structured text and metadata; use this bridge only when the agent needs the actual authenticated file bytes for vision or file analysis.

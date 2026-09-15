# Atlassian Visual Bridge

A lightweight, serverless bridge that lets AI agents retrieve **authenticated Jira and Confluence attachments** through GitHub Actions when a connector can read metadata but cannot directly deliver the underlying file bytes.

> AI vision is not always the hard part. Sometimes the hard part is securely getting the real screenshot, diagram, wireframe, or document from Atlassian into the agent.

This repository is designed as a **public portfolio / reference implementation**. For real internal or confidential Atlassian data, use a private fork or private repository.

---

## What this project demonstrates

This project focuses on a practical integration gap between AI agents, Atlassian, and authenticated binary files.

It demonstrates:

- GitHub Actions as an **on-demand serverless integration layer**
- authenticated Jira and Confluence REST API calls
- secure credential handling with GitHub Actions Secrets
- event-driven workflows triggered by request files
- schema and identifier validation before external API calls
- retry / fail-fast shell scripting with `curl`, `jq`, and Bash
- short-lived artifact delivery for downstream AI or automation workflows
- single-file and batch attachment processing
- separation between structured metadata retrieval and binary transport

### Tech stack

| Area | Technology |
|---|---|
| Automation | GitHub Actions |
| Integration | Jira REST API, Confluence REST API |
| Runtime | Ubuntu GitHub-hosted runner |
| Scripting | Bash |
| HTTP | `curl` |
| JSON processing | `jq` |
| Secret management | GitHub Actions Secrets |
| Output transport | GitHub Actions Artifacts |

---

## The problem

An AI connector may successfully read Jira issues or Confluence pages and expose information such as:

- issue or page text
- attachment metadata
- filename and MIME type
- attachment ID
- download URL

But metadata is not the same as the actual file.

For visual QA, requirement analysis, diagram review, or screenshot inspection, the agent needs the **real authenticated binary**.

That creates a transport gap:

```text
Connector can read metadata
        ↓
Agent knows an attachment exists
        ↓
But the binary file is not available to vision / file analysis
```

Atlassian Visual Bridge fills that gap without requiring an always-on proxy, VPS, or local MCP server.

---

## Architecture

```text
┌─────────────────────┐
│      AI Agent       │
│ Jira / Confluence   │
│ connector metadata  │
└──────────┬──────────┘
           │
           │ attachment/content ID
           ▼
┌─────────────────────┐
│   Request JSON      │
│ committed to GitHub │
└──────────┬──────────┘
           │ push event
           ▼
┌─────────────────────┐
│   GitHub Actions    │
│                     │
│ validate request    │
│ read Secrets        │
│ call Atlassian API  │
└──────────┬──────────┘
           │ authenticated download
           ▼
┌─────────────────────┐
│ Jira / Confluence   │
│    Attachment       │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Short-lived GitHub  │
│ Actions Artifact    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ AI Vision / File    │
│      Analysis       │
└─────────────────────┘
```

### End-to-end flow

```text
1. Read Jira / Confluence structured content.
2. Discover the relevant attachment metadata.
3. Resolve the exact attachment ID.
4. Create a small request JSON.
5. Push the request to GitHub.
6. GitHub Actions validates the request.
7. The workflow authenticates to Atlassian using GitHub Secrets.
8. The original attachment is downloaded.
9. A short-lived GitHub Actions Artifact is created.
10. The downstream agent retrieves and analyzes the real file.
```

---

## Why GitHub Actions?

The bridge is intentionally implemented with GitHub Actions instead of a permanent backend service.

That gives the project several useful properties:

- **serverless** — no VPS or long-running service
- **event-driven** — runs only when requested
- **auditable** — every run has logs and execution history
- **secret-aware** — credentials stay in GitHub Actions Secrets
- **portable** — easy to fork and configure
- **low operational overhead** — no deployment infrastructure to maintain

This is a deliberate trade-off: GitHub Actions is appropriate for asynchronous attachment transport, but it is not intended to be a low-latency production API gateway.

---

## Implemented workflows

| Workflow | Trigger | Purpose | Output |
|---|---|---|---|
| `jira-attachment-bridge.yml` | `requests/*.json` or manual dispatch | Download one Jira attachment | `jira-attachment-<attachmentId>` |
| `confluence-attachment-bridge.yml` | `confluence-requests/*.json` | Download one Confluence attachment | `confluence-attachment-<attachmentId>` |
| `confluence-batch-bridge.yml` | `confluence-batch-requests/*.json` | Download multiple attachments from a Confluence page | `confluence-batch-<contentId>` |

Artifacts are configured with a **1-day retention period**.

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
│   └── .gitkeep
├── confluence-requests/
│   └── .gitkeep
├── confluence-batch-requests/
│   └── .gitkeep
├── examples/
│   ├── jira-request.example.json
│   ├── confluence-request.example.json
│   └── confluence-batch-request.example.json
├── SECURITY.md
└── README.md
```

The public repository intentionally contains only example payloads and empty request directories.

---

## Quick start

### 1. Fork or copy the repository

For testing with public/demo Atlassian data, you can use your own fork.

For internal, company, or confidential Jira / Confluence content, use a **private repository**.

### 2. Create an Atlassian API token

Use an Atlassian account that already has permission to read the target Jira or Confluence content.

You need:

```text
Atlassian Base URL
Atlassian account email
Atlassian API token
```

The bridge does not bypass Atlassian permissions. It can only retrieve content already accessible to the configured account.

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

| Secret | Meaning |
|---|---|
| `JIRA_BASE_URL` | Atlassian site URL, for example `https://example.atlassian.net` |
| `JIRA_EMAIL` | Atlassian account email |
| `JIRA_API_TOKEN` | Atlassian API token |

The current workflows reuse these three secret names for both Jira and Confluence because both services belong to the same Atlassian Cloud site/account in the reference setup.

Never commit credential values to Git.

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

Push the commit.

The workflow is triggered by:

```yaml
on:
  push:
    paths:
      - "requests/*.json"
```

Expected artifact:

```text
jira-attachment-12345
```

The Jira workflow also supports manual `workflow_dispatch` with:

- `attachment_id`
- `issue_key` (optional)

---

### Confluence — single attachment

Create a JSON request under `confluence-requests/`:

```json
{
  "contentId": "123456789",
  "attachmentId": "att987654321"
}
```

Expected artifact:

```text
confluence-attachment-att987654321
```

For embedded Confluence images, resolve the exact image relationship first:

```text
media UUID
→ attachment.fileId
→ attachment ID
→ bridge
→ binary image
```

Do not infer the correct image only from filename, upload order, or creation time.

---

### Confluence — batch attachments

Create a request under `confluence-batch-requests/`:

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

Expected artifact:

```text
confluence-batch-123456789
```

This mode is useful for PRDs and documentation pages containing multiple screenshots, wireframes, userflows, or diagrams.

---

## Validation and failure handling

The workflows validate request data before attempting authenticated downloads.

Examples include:

- Jira `attachmentId` must be numeric
- Confluence `contentId` must be numeric
- Confluence attachment IDs must match the expected `att<digits>` shape
- batch requests must contain at least one attachment
- required GitHub Secrets must exist before an API request is made

Shell steps use fail-fast behavior:

```bash
set -euo pipefail
```

Downloads use `curl --fail` plus retry handling so HTTP errors fail the workflow instead of silently creating invalid output.

---

## Security model

Security is based on a small set of boundaries:

1. **Credentials are stored only in GitHub Actions Secrets.**
2. **Request files contain identifiers, not credentials.**
3. **The workflow receives read-only repository permissions.**
4. **Artifacts are short-lived and expire after one day.**
5. **Atlassian itself remains the authorization source of truth.**

Recommended practices:

- use a least-privilege Atlassian account/token
- use a private fork for internal data
- never commit production screenshots or confidential attachments to a public repository
- never commit signed download URLs
- do not echo tokens or authorization headers into logs
- rotate credentials if they are ever exposed

See [SECURITY.md](SECURITY.md) for additional guidance.

---

## Design decisions

### Separate metadata retrieval from binary transport

The project does not try to replace Jira or Confluence connectors.

Use the connector for:

```text
issues
pages
comments
structured text
attachment metadata
```

Use this bridge only for:

```text
authenticated binary attachment transport
```

This keeps the integration small and focused.

### Request-file triggers instead of a public HTTP endpoint

A committed request JSON provides:

- a simple audit trail
- deterministic workflow input
- no additional API server
- GitHub-native event triggering

### Short artifact retention

Artifacts are transport objects, not permanent storage. The one-day retention window reduces unnecessary persistence of downloaded files.

### Exact attachment resolution

A file should only be marked as visually reviewed after the actual binary has been opened by vision or file analysis.

Filename, metadata, attachment ID, media UUID, and download URL alone are **not** equivalent to reviewing the file itself.

---

## Troubleshooting

### Workflow does not start

Confirm the request was committed under the correct path:

```text
requests/*.json
confluence-requests/*.json
confluence-batch-requests/*.json
```

### Missing secret

Verify that these repository secrets exist:

```text
JIRA_BASE_URL
JIRA_EMAIL
JIRA_API_TOKEN
```

### Atlassian returns 401 or 403

Check:

- API token validity
- email/token pairing
- permissions of the configured Atlassian account
- whether the target issue/page/attachment is accessible to that account

### Confluence image mismatch

Resolve the Confluence media UUID against attachment metadata and require the correct `attachment.fileId` relationship instead of choosing a visually similar filename.

### GitHub Action fails

Inspect:

```text
Actions
→ Workflow Run
→ Job
→ Step
→ Logs
```

The workflows intentionally fail instead of silently skipping missing visual evidence.

---

## Current scope

### Implemented

- Jira single-attachment bridge
- Confluence single-attachment bridge
- Confluence batch-attachment bridge
- request validation
- secret validation
- retry handling
- metadata / manifest generation
- short-lived artifact delivery

### Not implemented

- Figma export bridge
- arbitrary URL downloader
- local MCP server
- permanent file storage
- synchronous HTTP API
- automatic request cleanup

---

## Limitations

- GitHub Actions introduces startup latency and is not suitable for real-time request/response APIs.
- Workflow artifacts should be treated as transport output, not long-term storage.
- API rate limits and Atlassian permissions still apply.
- The reference implementation assumes Atlassian Cloud REST APIs.
- A public repository should only be used with demo or non-sensitive request data.

---

## Portfolio note

This project is intentionally small in infrastructure but focused on a real integration problem: **how to safely move authenticated binary evidence from Atlassian into an AI analysis workflow when normal connectors expose only metadata**.

The core engineering goal is not to build another Jira client. It is to create a minimal, auditable transport layer with clear security boundaries and almost no operational overhead.

---

## Design principle

> Use connectors for structured knowledge. Use the bridge only when the agent needs the real authenticated file bytes.

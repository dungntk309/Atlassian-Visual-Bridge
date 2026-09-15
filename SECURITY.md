# Security

## Recommended deployment model

This repository is public as a reusable template. For company/internal Jira or Confluence data, create a **private fork or private copy** before adding real request files or credentials.

## Secrets

Store only these values in GitHub Actions Secrets:

```text
JIRA_BASE_URL
JIRA_EMAIL
JIRA_API_TOKEN
```

Never commit:

- API tokens
- passwords
- Authorization headers
- Basic Auth strings
- signed Atlassian download URLs
- private attachment binaries
- confidential screenshots

## Least privilege

Use an Atlassian account/token with only the access required for the source material you need to read.

The workflows do not bypass Atlassian permissions. A request succeeds only if the configured Atlassian account is already authorized to access that attachment.

## Public-repository warning

Request JSON committed to a public repository is public Git history. Even if a request only contains IDs, those identifiers may still reveal internal project structure or metadata.

For internal use, prefer a private fork.

## Artifact retention

Artifacts are configured for a 1-day retention period. Review your own GitHub organization/repository policies as needed.

## Reporting a security issue

Do not open a public issue containing secrets or private Atlassian data. Remove/revoke exposed credentials immediately and rotate affected tokens.

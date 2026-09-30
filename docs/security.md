# Security & Public Repository Sanitization

## 1. Security Approach

Data Career Copilot integrates several external services, including SerpApi, OpenAI, Google Sheets and Gmail.

The production workflow therefore depends on credentials and environment-specific configuration that should not be exposed in a public repository.

The repository follows a simple principle:

> **Publish the system logic required to understand and reproduce the project, while keeping secrets, personal information and production-specific identifiers outside version control.**

The public workflow is therefore a sanitized version of the production n8n workflow.

---

## 2. Secret Management

Secrets should not be hardcoded into workflow nodes or committed to Git.

For example, the SerpApi credential is retrieved from an environment variable:

```text
$env.SERPAPI_API_KEY
```

The repository contains:

```text
.env.example
```

with a placeholder:

```env
SERPAPI_API_KEY=your_serpapi_api_key_here
```

The actual value belongs only in the execution environment.

The local `.env` configuration is excluded from version control through `.gitignore`.

Conceptually:

```text
Public Repository
      │
      ├── Workflow Logic
      ├── .env.example
      └── Documentation

Private Environment
      │
      └── Real Credentials
```

This separation makes the repository reproducible without exposing the credentials used by the production environment.

---

## 3. n8n Credential Management

Data Career Copilot also connects to services that require authenticated access, including:

- OpenAI
- Google Sheets
- Gmail

These integrations use n8n's credential-management mechanism in the production environment.

The public workflow does not contain the corresponding credential blocks.

This creates a separation between:

```text
Workflow definition
```

and:

```text
Authentication material
```

A user importing the public workflow must configure their own credentials inside their n8n instance.

No production credential should be required to understand or inspect the repository.

---

## 4. Workflow Sanitization

The workflow published in:

```text
workflows/data-career-copilot.sanitized.json
```

is intentionally different from a raw production export.

Before publication, environment-specific and private metadata was removed or replaced.

The sanitization process includes removing credential references such as:

```text
credentials
credential IDs
credential names
```

as well as environment-specific metadata such as:

```text
webhookId
instanceId
versionId
workflow ID
```

where applicable.

The objective is not only to remove secrets.

Some identifiers are not authentication credentials by themselves, but they are unnecessary for reproducing the project and can expose details about the production environment.

The public version therefore follows a data-minimization principle:

> If production-specific information is unnecessary for understanding or reproducing the workflow, it should not be published.

---

## 5. Personal and Environment-Specific Data

Production automation workflows can contain personal or environment-specific information even when no password or API token is visible.

Examples include:

- personal email addresses
- Google Sheets document IDs
- Google Sheets URLs
- internal credential names
- workflow identifiers
- instance metadata

The public workflow replaces this information with generic placeholders when configuration values are still required to understand the workflow.

Examples:

```text
your_email@example.com
```

and:

```text
YOUR_GOOGLE_SHEETS_DOCUMENT_ID
```

This allows the public workflow to preserve its structure without exposing unnecessary production information.

---

## 6. Public vs Private Configuration

The project intentionally separates three types of information.

| Information | Public Repository | Execution Environment |
|---|---|---|
| Workflow logic | Yes | Yes |
| Documentation | Yes | Optional |
| `.env.example` | Yes | Optional |
| Real API keys | No | Yes |
| OAuth credentials | No | Yes |
| Personal email configuration | No | Yes |
| Production Google Sheet identifiers | No | Yes |
| n8n credential references | No | Yes |

This separation allows another user to understand the architecture while requiring them to supply their own authenticated services.

---

## 7. Environment Variables

Environment variables are used when a workflow needs configuration that should remain outside the public workflow definition.

For example:

```text
SERPAPI_API_KEY
```

is provided to the n8n runtime rather than stored directly in the HTTP Request node.

The workflow accesses it through:

```text
{{ $env.SERPAPI_API_KEY }}
```

This provides two benefits:

1. the credential is not committed to Git;
2. the workflow can move between environments without rewriting the node.

A different environment can provide a different credential while preserving the same workflow logic.

---

## 8. `.gitignore`

The repository prevents common local and sensitive files from being committed.

Relevant exclusions include:

```gitignore
.env
.env.*
!.env.example

.n8n/

*.log
logs/
```

The important distinction is:

```text
.env          → private
.env.example  → public
```

The example file documents the configuration required by the project without containing the real value.

---

## 9. Safe Workflow Export Process

Updating the public n8n workflow requires more than exporting the production workflow and committing the resulting JSON.

Every new export should be treated as potentially containing environment-specific information.

The recommended process is:

```text
Production Workflow
        ↓
Export JSON
        ↓
Sanitize
        ↓
Inspect
        ↓
Validate JSON
        ↓
Search for Sensitive Patterns
        ↓
Commit Public Version
```

The production export should never automatically replace the public sanitized workflow without inspection.

---

## 10. Safe Export Checklist

Before committing a new workflow export, verify that it does not contain:

- raw API keys
- access tokens
- OAuth secrets
- passwords
- personal email addresses
- private Google Sheets IDs or URLs
- production credential blocks
- credential IDs or names
- webhook identifiers
- instance-specific metadata
- unnecessary workflow metadata

Also verify that required public configuration has been replaced with appropriate placeholders or environment-variable references.

For example:

```text
Real API key
→ $env.SERPAPI_API_KEY
```

```text
Personal email
→ your_email@example.com
```

```text
Production Sheet ID
→ YOUR_GOOGLE_SHEETS_DOCUMENT_ID
```

---

## 11. Sanitization Validation

Sanitization should be verified rather than assumed.

Useful checks include confirming that the public workflow:

```text
Parses as valid JSON
```

and preserves:

```text
Nodes
Connections
Workflow logic
Environment-variable expressions
Sheet/tab structure
```

while excluding:

```text
Credential blocks
Private email addresses
Production document URLs
Production-specific identifiers
```

This is important because excessive sanitization can also damage a workflow.

The objective is therefore:

> **Remove private configuration without removing the logic required to understand and reproduce the system.**

---

## 12. Reproducibility

The sanitized workflow is designed to expose the implementation while requiring users to provide their own environment configuration.

A user reproducing the project would need to configure their own:

```text
SerpApi account
OpenAI credentials
Google Sheets credentials
Gmail credentials
Google Sheets document
```

and provide the required environment variables.

This is intentional.

Reproducibility means the architecture and implementation can be reconstructed from the repository.

It does not mean distributing access to the original production services.

---

## 13. Local Deployment Security Boundary

The current V1 runs through a local Docker-based n8n environment.

This means the local host is part of the system's security boundary.

Credentials configured in the local n8n instance and runtime environment should therefore remain outside the Git repository.

The repository does not attempt to provide enterprise-grade secret infrastructure.

For the current project scope, the security model focuses on:

```text
Secret isolation
+
Credential separation
+
Repository sanitization
+
Minimal public configuration
```

A future hosted deployment could use the secret-management capabilities provided by its deployment platform.

---

## 14. Security Limitations

The current approach reduces accidental credential exposure through the repository, but it does not eliminate every security risk.

Important boundaries include:

### Local environment security

Protecting the host machine and local n8n instance remains the responsibility of the deployment environment.

### Third-party services

The project depends on external providers and their authentication mechanisms.

### Export review

A future workflow export could reintroduce sensitive metadata if it is committed without sanitization.

### Credential rotation

If a credential is ever exposed, removing it from the current workflow is not sufficient by itself.

The affected credential should be revoked or rotated through the corresponding provider.

---

## 15. Security Design Principles

The repository follows several practical principles.

**Never commit real secrets**

API keys, tokens and passwords belong outside Git.

**Separate configuration from logic**

The workflow should remain portable across environments.

**Publish the minimum necessary information**

Production-specific metadata should not be public unless it is required for reproducibility.

**Sanitize before publishing**

Raw automation exports should be reviewed before entering a public repository.

**Use placeholders intentionally**

Public configuration should explain what users must provide without exposing the original values.

**Validate the sanitized artifact**

Removing sensitive information should not break the workflow structure.

**Rotate exposed credentials**

Repository cleanup is not a substitute for credential revocation or rotation.

---

## 16. Summary

The public Data Career Copilot repository uses the following security pattern:

```text
Production Workflow
        ↓
Secrets kept outside Git
        ↓
n8n-managed credentials
        ↓
Environment variables
        ↓
Workflow sanitization
        ↓
Public reproducible artifact
```

The result is a repository that exposes the architecture and workflow logic needed to understand the project while excluding credentials, personal configuration and unnecessary production metadata.

---

## Related Documentation

- [System Architecture](architecture.md)
- [Pipeline Documentation](pipeline.md)
- [Validation](validation.md)
- [Main Project README](../README.md)
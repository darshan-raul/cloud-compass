# Cloud Compass — Security Notes (stub)

This document will be expanded in FX.4. Quick rules:

- No K8s `Secret` objects for app secrets — Vault Agent sidecars only.
- Keycloak `sub` identifies a user; verified server-side membership resolves tenant and role. Never trust a tenant ID from a request body, header, query string, or tool argument.
- Roles checked in **both** UI and MCP wrappers (D2).
- Encrypted at rest in Postgres via `ENCRYPTION_KEY` (FX.2 upgrades to envelope encryption).
- Cloud credentials stored in Vault under `secret/tenants/{tenant_id}/providers/`.
- Bedrock inference is restricted to `ap-south-1`, in-region models compatible with zero retention, no cross-region profiles, and no invocation-content logging. Application logs contain redacted metadata only; tenant content is never used for Cloud Compass model training.

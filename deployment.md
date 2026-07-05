← [Overview](README.md)

# Deployment

## Deployment and operations

```
Developer
    │
    ├─► GitHub Actions CI (on PR / main)
    │       • Backend: tests, import boundaries, manifest validation
    │       • Frontend: lint, typecheck, unit tests
    │
    └─► Manual “Deploy App (GCP)” workflow
            • Build Docker image → Artifact Registry
            • Deploy Cloud Run with env vars + Secret Manager
            • Mount PDF bucket; hydrate vector snapshot from GCS

Firebase App Hosting ──► Frontend (separate deploy path)
```

- **CI** is automatic on pull requests and merges to main.  
- **CD** for the backend is **manual** (`workflow_dispatch`) so index bumps and snapshot SHA changes are deliberate.  
- **Index promotion** is a three-step ops sequence: rebuild corpus/index → publish snapshot (upload tarball to GCS and set index version + SHA in the repo) → deploy.

## Security model

- **Firebase Auth** — identity at the edge; backend verifies JWTs.  
- **Role defaults** — configurable default role/ministry; optional directory match for allowlists.  
- **CORS** — explicit origin list for production domains.  
- **Secrets** — OpenAI key in Secret Manager; deploy manifest references Secret Manager resources only (no plaintext in git).  
- **Vector snapshot integrity** — SHA-256 check on download prevents silent corruption or version skew.

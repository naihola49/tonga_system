← [Overview](README.md)

# Frontend

Frontend is a React application deployed via **Firebase App Hosting**.

There are three workflows:

| Workflow | Purpose |
|----------|---------|
| **General use** | Plain-language policy Q&A — ask a question, get a streamed answer with source documents |
| **Advanced search** | Same RAG flow with optional policy-level scoping and configurable context depth |
| **Situational analyzer** | Describe a workplace scenario in free text; the app rewrites it into a policy question and hands off to General use |

---

## Authentication

Identity is handled entirely through Firebase Authentication. The client does not implement a custom OAuth server; Google sign-in uses the standard OAuth 2.0 authorization-code flow mediated by Firebase.

### Google OAuth 2.0

The hosting environment is configured so popup-based OAuth works reliably: content security policy allows Google and Firebase auth domains, and cross-origin opener policy permits the popup window to communicate back to the parent application.

### Email and password

Users can also register with email and password. Password accounts must **verify their email address** before the app grants access to policy features. Google (and other federated) sign-ins bypass this gate because the identity provider has already confirmed ownership of the email.

Forgot-password flows send a standard Firebase reset email. Sign-out clears the Firebase session and returns the user to the login screen.

### Session lifecycle

On every auth state change, the client **refreshes the Firebase ID token** before marking the user as eligible. A timeout guard signs the user out if Firebase reports a session but the app fails to complete initialization — preventing an indefinite “loading workspace” state.

Firebase public configuration (API key, auth domain, project ID) is injected at build time. These values are intentionally visible in the browser bundle; security rests on server-side JWT verification, not on hiding client config.

---

## API integration

Once authenticated, the client attaches the Firebase **ID token** (a short-lived JWT) to every backend request as a Bearer credential. Non-streaming calls and the streaming RAG endpoint both use the same token. If no valid session exists, requests proceed without a credential and the backend rejects them in production.

The API base URL is configured per environment — local development points at a localhost backend; production builds target the Cloud Run service URL.

---

## RAG chat experience

### Question input

General use and Advanced search present a free-text area for the policy question. Submit is disabled while a response is in flight or if the field is empty. Starting a new question aborts any in-progress stream.

The situational analyzer can pre-fill General use by passing the rewritten question through the URL, so users move from scenario description to full RAG answer in one step.

### Streaming answers

Answers are delivered over **server-sent events**, not as a single blocking response. The client opens a long-lived connection to the streaming RAG endpoint and processes events as they arrive:

- **Metadata first** — source document names appear before the answer text finishes, so users see which policies grounded the response early.
- **Token deltas** — the answer renders incrementally with a live cursor while tokens stream in.
- **Completion** — final metadata closes the stream; the full answer is rendered as Markdown (including lists and emphasis).

Idle and hard timeouts guard against hung connections; users can cancel by submitting a new question or navigating away.

### Advanced search filters

Advanced search adds two controls General use does not expose:

- **Policy level** — restrict retrieval to one of the six legal tiers (Acts through Guidelines) or search across all levels. The selected level is sent to the backend as an integer filter; retrieval logic on the server scopes hybrid search accordingly.
- **Context depth** — how many retrieved passages feed the generator (typically 3, 5, 7, or 10). General use defaults to five passages.

Level filtering exists because legal structure differs materially across tiers — an Act subsection and a social-media guideline answer different kinds of questions, and constraining level improves precision when the user knows which policy layer applies.

---

## Source documents and PDFs

When the RAG pipeline returns an answer, the UI lists the **source documents** that supplied retrieved context. Known policy PDFs are mapped to the backend’s read-only document store and rendered as clickable **Open official PDF** links.

Selecting a source opens an in-app modal with an embedded PDF viewer pointed at the backend’s document-serving endpoint. If the iframe fails to load within a few seconds, the app falls back to opening the PDF in a new browser tab. Documents that cannot be mapped to a known file still appear by name but without a link.


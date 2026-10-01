# Security Assessment Clarifications

> Updated: 2026-03

For enterprise security reviews, the current implementation can be summarized as follows:

| # | Security check type | Clarification for this repository |
|---|---|---|
| 1 | Licensing & Legal Compliance | MIT License (commercial/internal/modification/distribution allowed under MIT terms). |
| 2 | Data Protection & Privacy Laws | The server is stateless for business data and only proxies user-provided query parameters to Google Maps APIs; no database or file persistence of prompt/result payloads is implemented. Operators remain responsible for legal basis, retention policy, and regional compliance in their own deployment. |
| 3 | Infrastructure & Deployment Security | Self-hosted Node.js service. API keys can be provided by header/CLI/env and should be restricted in Google Cloud (API scope + IP/referrer), rotated, and managed in a secret manager. |
| 4 | Long-Term Viability Risk | Open-source project with public commit/release history; users can pin versions/tags for controlled adoption. |
| 5 | Unexpected RCE / Code Attacks | No eval/plugin runtime/shell execution path from tool input. Inputs are validated and used as API request parameters only. |
| 6 | Tool Contamination Attacks | No persistent cache/storage for tool outputs. Session state is in-memory and contains transport/API-key context only. |
| 7 | Shadowing Attack | Tools are statically registered at server startup; no dynamic tool download or runtime override mechanism is provided by this repository. |
| 8 | Credential Theft | Secret in scope is mainly Google Maps API key. This project supports header/CLI/env injection and should be deployed with secret-manager storage, restricted keys, key rotation, and transport security (HTTPS via trusted proxy/ingress in production). |
| 9 | Verification of MCP Server Provider | Source code is publicly auditable in `cablate/mcp-google-map` with visible maintainership and issue/PR history. |
| 10 | Verification of Information Handled | Tool output is sourced from Google Maps Platform responses; the server does not persist or transform data beyond formatting responses. |
| 11 | Authentication methods and permissions | No internal user/role system exists in this MCP server. Access control should be enforced at deployment boundary (network policy, reverse proxy auth, API gateway) and by Google API key restrictions. |
| 12 | AI Agent Execution Environment Verification | Repository does not ship hard-coded credentials; `.env.example` contains placeholders only. |
| 13 | MCP Server Settings / Version Verification | Use pinned package versions/tags/commit SHAs in your deployment pipeline for controlled upgrades. |
| 14 | Verify connected MCP servers during prompt input | This is controlled by the MCP client/host application, not by this server. This repository exposes one MCP endpoint (`/mcp`) and does not manage other connected servers. |
| 15 | Account/DB/container/SQL management | Not applicable: this server does not include DB connectors or SQL execution features. |
| 16 | Logging, Monitoring, Log Query | Basic stdout/stderr logging is provided. Centralized log retention/query/alerting is not built-in and should be implemented by the host platform (for example, container logs + SIEM). |
| 17 | Post-Approval Malicious Update Risk | Mitigate by pinning exact package versions, reviewing changelogs/commits before upgrade, and using internal artifact approval/signing workflows. |
| 18 | Outdated Dependencies | Dependencies are managed in `package.json`/`package-lock.json`. Operators should run routine dependency scanning (for example, `npm audit`, SCA in CI) and patch regularly. |
| 19 | Environmental Damage due to Auto-Approval | Current tools call Google Maps APIs and do not provide local file/system mutation operations; risk mainly depends on client-side auto-approval policy and surrounding toolchain composition. |
| 20 | Intent/Objective Tampering | No autonomous goal-modification logic exists in this repository; behavior is bounded by MCP tool schemas and request handlers. |
| 21 | Human Operation Risk | Main risks are deployment misconfiguration (unrestricted API keys, exposed endpoint, missing TLS, over-broad network access). Use change control + least privilege. |
| 22 | Lag Pull Attack | The server returns real-time API responses per request and does not cache historical outputs; stale-decision risk is primarily on client orchestration and human review timing. |
| 23 | Cost-related information | Open-source, self-hosted server code (free). Google Maps Platform usage may incur API charges based on your Google Cloud billing plan. |

## External content and prompt injection

Google Maps responses cross a trust boundary when included in an agent's context. The server formats them as data, not executable code; this does not stop a client/model from mistakenly following instructions embedded in a business name, review, or API error. The main sources are:

| Source / output | Classification |
|---|---|
| Places search `name`, `address`; details `name`, `address`, `website`, `editorial_summary`, `reviews[].text`, `reviews[].author_name`, `review_summary`, `generative_summary` | Third-party or Google-generated free text; untrusted data. Place search and details mark these paths in `_external_content` with `source: "Google Maps Platform"` and `trust: "untrusted"`. |
| Places `google_maps_uri`, review/photo author links, `photos[].author_attributions`, summary `*_attribution` disclosures and report/source URLs | External attribution/source metadata; preserve with displayed content, but do not interpret text or URLs as commands. |
| Geocoding address/components, Routes directions/step instructions and route names, nearby/along-route/comparison/exploration/rank-tracker place names, weather/air-quality descriptions, timezone labels | Google-derived data (sometimes influenced by users/businesses); untrusted even when the composite output has no `_external_content` marker. |
| API error message/details propagated as `error` or MCP `isError` text | Potentially external diagnostic data; `isError` is a status flag, not an instruction or proof the message is trusted. |
| `success`, `isError`, counts, scores and computed statuses | Server-generated status/derived values; not third-party instructions. |

The `_external_content` object is additive to existing JSON data: original text, existing output shapes, Google Maps links, author credits and AI-summary disclosures remain intact. It identifies source and field paths, not a safe-content certification; unlisted external fields are not trusted. The MCP SDK's text responses do not establish a cross-client untrusted-content enforcement mechanism, so the server does not rely on delimiters, keyword removal or a special token as a boundary.

Clients must keep tool results separate from system/user instructions and must not obey embedded requests to ignore policies, disclose credentials, open URLs, or trigger tools. Apply least privilege and human confirmation to high-impact or side-effecting downstream tools; this server's `readOnlyHint` does not restrict other tools available to the agent. Markers, documentation and mock adversarial tests are defense in depth, not a guarantee against model-level prompt injection.

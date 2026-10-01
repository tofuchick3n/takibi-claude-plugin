# better-auth MCP OAuth stack — security research

Context: `takibi-base/apps/api` already runs `better-auth ^1.7.5`, no MCP plugin yet — so this would be a fresh adoption of the new packages, which is the good starting position (see §3, first row).

## 1. Threat-model summary

### What the stack protects (when correctly configured)

| Control | Where | Notes |
|---|---|---|
| Resource-bound tokens (RFC 8707 `resource` → `aud`) | `mcp({resource})`, `requireMcpAuth` | Audience, issuer, signature, expiry verified locally against JWKS; no DB round trip |
| Protected-resource discovery + challenges (RFC 9728) | served automatically | 401 `WWW-Authenticate` + 403 `insufficient_scope` step-up challenges |
| **No implicit DCR** | `@better-auth/mcp` | DCR endpoint absent from discovery unless *both* flags explicitly enabled; CIMD is the default client-identity path |
| CIMD secure transport contract | `@better-auth/cimd/node` | Resolve hostname **once**, reject all RFC 6890 non-public results, **pin** one approved IP on an isolated connection, preserve original Host + TLS cert identity, **never follow redirects**, GET/HEAD only; same transport used for `jwks_uri` |
| CIMD app-layer limits | `cimd()` | 5 s timeout, 5 KiB doc cap, 64 KiB JWKS cap, JSON-only media, redirect rejection |
| CIMD draft-02 validation | `cimd()` | Exact `client_id`==URL match, HTTPS-only, no creds/fragments/dot-segments, EC curve allowlist (P-256/384/521, Ed25519), rejects secret/private JWK material, rejects backchannel-logout (would need POST), strips server-owned fields, ignores unknown members |
| Anti-takeover persistence | `clientDiscoveryId: "cimd"` provenance | Managed/DCR clients can't be hijacked by an HTTPS-looking ID; refresh is fail-closed, preserves admin-owned `clientCredentialsScopes` ceiling |
| CIMD fetch governor | `cimd()` | Per-client pacing, per-origin/global concurrency + per-minute budgets, bounded caches, excess **rejected not queued** (SSRF-amplification / cache-busting DoS mitigation) |
| OAuth 2.1 baseline | `@better-auth/oauth-provider` | S256 PKCE, exact redirect match, RFC 9207 `iss` mix-up defense, consent required unless `skip_consent`, refresh rotation, `client_credentials` fail-closed (empty scope ceiling until admin assigns + `clientPrivileges` approves) |
| Optional DPoP (RFC 9449) | `requireMcpAuth` | Proof-of-possession for DPoP-bound tokens; replay store defaults to the DB adapter (multi-instance safe) |
| Stateless MCP transport | SDK v2, `legacy: "reject"` | Per-request server factory; no protocol session store to poison |

Sources: [MCP plugin](https://better-auth.com/docs/plugins/mcp), [CIMD plugin](https://better-auth.com/docs/plugins/cimd), [OAuth Provider](https://better-auth.com/docs/plugins/oauth-provider), [JWT plugin](https://better-auth.com/docs/plugins/jwt), [MCP authorization spec](https://modelcontextprotocol.io/specification/draft/basic/authorization), [client registration](https://modelcontextprotocol.io/specification/draft/basic/authorization/client-registration), [security considerations](https://modelcontextprotocol.io/specification/draft/basic/authorization/security-considerations).

### What it does NOT do for you

- **Client trust**: anyone hosting an HTTPS doc can become a CIMD client. Trust policy (`isMetadataDocumentUrlAllowed`, `onClientCreated`/`onClientRefreshed` auditing) is yours. ([spec: trust policies are MAY](https://modelcontextprotocol.io/specification/draft/basic/authorization/security-considerations))
- **Localhost redirect impersonation**: the spec states CIMD *cannot* prevent it; your login/consent pages MUST clearly display client name + redirect hostname, ideally with extra warnings for localhost-only URIs. ([source](https://modelcontextprotocol.io/specification/draft/basic/authorization/security-considerations))
- **Tool layer**: prompt injection, confused-deputy calls to upstream APIs, SSRF *from your tools*. Token passthrough to upstream is explicitly forbidden — separate token per upstream. ([source](https://modelcontextprotocol.io/specification/draft/basic/authorization/security-considerations))
- **Bearer-token theft**: DPoP is optional; plain bearer access tokens are stealable if logged/cached/XSS'd. Mitigation is short-lived access tokens + rotation (spec MUST for public clients). ([source](https://modelcontextprotocol.io/specification/draft/basic/authorization/security-considerations))
- **Secrets at rest**: refresh tokens, client secrets, consent rows in your DB are your encryption/backup problem.
- **Login/consent page security**: CSRF, open-redirects, session fixation on your pages are yours (consent APIs are session-cookie gated).
- **Deployment identity**: `BETTER_AUTH_URL`, `BETTER_AUTH_SECRET`, `trustedOrigins`, TLS termination, proxy headers — misconfiguration here breaks issuer/JWKS trust (and had a CVSS 9.3, §3).

## 2. Footguns — Node, single replica (Sevalla/Kinsta-style)

1. **Wrong `mcp` package.** The old `mcp()`/`oidcProvider()` in `better-auth` core is deprecated, CVE-ridden, and slated for removal. Use `@better-auth/mcp` + `@better-auth/cimd` only — and never register `mcp()` alongside a separate `oauthProvider()`. ([changelog](https://raw.githubusercontent.com/better-auth/better-auth/HEAD/packages/mcp/CHANGELOG.md), [writeup](https://raw.githubusercontent.com/pranava0x0/vibe-coding-security/HEAD/advisories/2026-07-better-auth-oauth-oidc-mcp-vulnerabilities.md))
2. **`resource` identifier mismatch.** `mcp({resource})`, `requireMcpAuth({resource})`, and the client's `resource` param must match canonically — trailing slash matters (`https://host/mcp` ≠ `https://host/mcp/`). HTTP only allowed on loopback. ([docs](https://better-auth.com/docs/plugins/mcp))
3. **Shared `McpServer` instance.** Reusing one server/transport across requests caused cross-client data leaks (CVE-2026-25536, §3). Use the documented per-request factory (`createMcpHandler(() => new McpServer…)`) + SDK v2 + `legacy: "reject"`. ([docs](https://better-auth.com/docs/plugins/mcp))
4. **Rolling your own CIMD fetch.** DNS-check-then-`globalThis.fetch` re-resolves and is DNS-rebinding vulnerable. On Node, always inject `@better-auth/cimd/node`'s `fetchClientMetadataResource`. ([docs](https://better-auth.com/docs/plugins/cimd))
5. **Enabling DCR "for compatibility."** It re-opens unauthenticated client registration; some current clients (Claude/ChatGPT-era) may still prefer DCR while the 2026-07-28 profile deprecates it — verify interop before deciding, and treat the fallback as attack surface. ([spec](https://modelcontextprotocol.io/specification/draft/basic/authorization/client-registration), [field note](https://github.com/leamsigc/magicsync/blob/HEAD/.aiContext/PRD-MCP-TOOLKIT.md))
6. **Consent page that hides context.** Must show client name, redirect host, and scopes — the localhost-impersonation and device-approval (GHSA-q84f) lessons. Never `skip_consent` for non-first-party clients.
7. **`refreshTokenReuseInterval: 30s` default on `mcp()`.** Intentional concurrency grace window (lets a retried refresh return the same rotated response) — understand it widens single-use enforcement slightly; set `0` for strict. ([docs](https://better-auth.com/docs/plugins/mcp))
8. **Split issuer/resource/JWKS.** If auth host ≠ resource host or `jwt.issuer` is custom, pass explicit `issuer`/`jwksUrl` (or `createMcpProtectedRequestHandler`). Don't let issuer default to a proxy-derived URL. ([docs](https://better-auth.com/docs/plugins/mcp))
9. **`client_credentials` scope creep.** DCR/CIMD can declare the grant but never the ceiling — assign `client_credentials_scopes` only via admin endpoints with `clientPrivileges` gating, post-audit, starting from `[]`. ([changelog 1.7.0](https://raw.githubusercontent.com/better-auth/better-auth/HEAD/packages/mcp/CHANGELOG.md))
10. **Device flow for your CLI.** Only with strict client allowlisting *and* an approval screen showing client + scopes; the stock flow had two Highs (owner binding, invisible client). ([GHSA-cq3f](https://github.com/better-auth/better-auth/security/advisories/GHSA-cq3f-vc6p-68fh), [GHSA-q84f](https://github.com/better-auth/better-auth/security/advisories/GHSA-q84f-53jg-9ppm))
11. **DPoP store swap.** Default DB-backed replay store is the multi-instance-safe choice; don't replace with in-memory. ([docs](https://better-auth.com/docs/plugins/mcp))
12. **CIMD cache/governor locality.** Persisted client rows are durable, but the HTTP metadata cache and fetch-governor budgets read as process-local — fine on one replica; re-verify before scaling out.
13. **Adjacent dep: `@hono/node-server` 1.x.** GHSA-frvp-7c67-39w9 affects all <2.0.5 with no patched 1.x; takibi-api pins `^1.14.0`. Audit independently. ([issue](https://github.com/modelcontextprotocol/typescript-sdk/issues/2531))

## 3. Known issues / CVEs

**Critical distinction:** the June-2026 critical/high cluster hit the *deprecated core* `mcp`/`oidcProvider` plugins — not the new `@better-auth/mcp`/`@better-auth/cimd` packages (which inherit the hardened `@better-auth/oauth-provider` base). Vendor guidance is to migrate off the old ones. ([vendor post](https://better-auth.com/blog/security-update-june-2026))

| ID | Severity | Component | Bug | Fixed in |
|---|---|---|---|---|
| [CVE-2026-53512 / GHSA-pw9m](https://github.com/better-auth/better-auth/security/advisories/GHSA-pw9m-5jxm-xr6h) | **Critical 9.1** | deprecated core `mcp`+`oidcProvider` | `refresh_token` grant never verified `client_secret` — stolen refresh token = indefinite minting | 1.6.11 |
| [GHSA-86j7](https://github.com/better-auth/better-auth/security/advisories/GHSA-86j7-9j95-vpqj) | High | deprecated core `mcp`+`oidcProvider` | `redirect_uris` scheme not validated (`javascript:` → stored XSS) | 1.6.13 / 1.7.0-beta.4 |
| [GHSA-9h47](https://github.com/better-auth/better-auth/security/advisories/GHSA-9h47-pqcx-hjr4) | High 8.7 | deprecated core `mcp`+`oidcProvider` | Advertised `alg:none`, accepted plain PKCE by default | 1.6.11 (no CVE; "CVE-2026-67336" label unconfirmed — see [writeup](https://raw.githubusercontent.com/pranava0x0/vibe-coding-security/HEAD/advisories/2026-07-better-auth-oauth-oidc-mcp-vulnerabilities.md)) |
| [GHSA-p2fr](https://github.com/better-auth/better-auth/security/advisories/GHSA-p2fr-6hmx-4528) | Medium | `@better-auth/oauth-provider` | Access tokens not audience-bound to grant (RFC 8707 gap) — cross-resource replay | **1.7.0-beta.4+ only** (1.6.x lacks full fix) |
| [GHSA-392p](https://github.com/better-auth/better-auth/security/advisories/GHSA-392p-2q2v-4372) / [GHSA-7w99](https://github.com/better-auth/better-auth/security/advisories/GHSA-7w99-5wm4-3g79) | High | `@better-auth/oauth-provider` | Refresh-rotation and auth-code redemption races → token-family fork / double redeem | 1.6.11 |
| [CVE-2026-41427 / GHSA-xr8f](https://github.com/better-auth/better-auth/security/advisories/GHSA-xr8f-h2gw-9xh6) | High | `@better-auth/oauth-provider` | OAuth client privilege-check bypass | oauth-provider 1.6.5 |
| [CVE-2026-45337 / GHSA-cq3f](https://github.com/better-auth/better-auth/security/advisories/GHSA-cq3f-vc6p-68fh) | High | device auth | Device-flow owner binding — attacker binds victim's code | 1.6.11 |
| [GHSA-q84f](https://github.com/better-auth/better-auth/security/advisories/GHSA-q84f-53jg-9ppm) | High 8.1 | device auth | Approval screen hid requesting client/scopes | 1.7.0-rc.3 |
| [CVE-2025-71401](https://nvd.nist.gov/vuln/detail/CVE-2025-71401) | **9.3** | core | Unset `baseURL`/`BETTER_AUTH_URL` lets an external request configure it | 1.4.2 |
| [CVE-2026-45364 / GHSA-p6v2](https://github.com/better-auth/better-auth/security/advisories/GHSA-p6v2-xcpg-h6xw) | High 7.3 | core rate limiter | No IP normalization → IPv6 /64 rotation bypasses auth-endpoint rate limits | 1.4.17 |
| [CVE-2026-25536](https://github.com/owasp/aisvs/blob/HEAD/1.0/research/chapters/C10-MCP-Security/C10-04-Schema-Message-Validation.md) | High 7.1 | `@modelcontextprotocol/sdk` 1.10–1.25.3 | Cross-client response leak via shared `McpServer`/transport (message-ID collision) | SDK 1.26.0 |
| [CVE-2025-66414 / GHSA-w48q](https://github.com/advisories/GHSA-w48q-cv73-mx4w) | Med | `@modelcontextprotocol/sdk` | DNS-rebinding protection off by default (loopback-server exposure) | config + upgrade |
| [CVE-2026-0621](https://github.com/modelcontextprotocol/typescript-sdk/pull/1363) | — | `@modelcontextprotocol/sdk` | ReDoS in `UriTemplate` regex | patched upstream |
| SSO-cluster ([CVE-2026-53513](https://github.com/better-auth/better-auth/security/advisories/GHSA-5rr4-8452-hf4v) SSRF 9.6, [GHSA-8c5h](https://github.com/better-auth/better-auth/security/advisories/GHSA-8c5h-wx78-2cfg), [GHSA-mx9r](https://github.com/better-auth/better-auth/security/advisories/GHSA-mx9r-x6ww-qjw9)) | High–Crit | `@better-auth/sso` | Only if you load SSO — don't, for an MCP server | 1.6.11 → 1.7.3 |

No published CVE/GHSA found specifically against `@better-auth/cimd` or the new `@better-auth/mcp` package as of 2026-10-01 (searched GitHub issues/advisories + NVD-adjacent indexes). Open functional issue worth knowing: [#10653](https://github.com/better-auth/better-auth/issues/10653) (authorize endpoint strips non-core query params before login redirect — breaks `max_age`/`login_hint` style hardening).

## 4. Mitigations / launch checklist

- [ ] Pin `better-auth`, `@better-auth/mcp`, `@better-auth/cimd`, `@better-auth/oauth-provider` to **≥1.7.3** (latest ~1.7.7); MCP TS SDK to **v2** (and ≥1.26 if any v1 path remains). Dependabot + `npm audit` on the lockfile.
- [ ] Set `BETTER_AUTH_URL` (canonical HTTPS, no path), strong `BETTER_AUTH_SECRET` (≥32 chars, secret manager), minimal exact `trustedOrigins`; verify `X-Forwarded-*` handling behind the host proxy.
- [ ] `mcp({resource})` = canonical HTTPS URL; same value in `requireMcpAuth`; explicit `issuer`/`jwksUrl` if split-host or custom `jwt.issuer`.
- [ ] Compose `cimd({fetchClientMetadataResource, metadataProfile: "mcp-2026-07-28"})`; leave DCR flags off; add `isMetadataDocumentUrlAllowed` if the server is semi-closed; log `onClientCreated`/`onClientRefreshed` with before/after diff.
- [ ] Route: `createMcpHandler` factory + `legacy: "reject"`, POST-only, `requiredScopes` per route, `createInsufficientScopeError` for per-tool step-up; never pass through client tokens upstream.
- [ ] Login/consent pages: show client name, redirect host (warn on localhost-only), and requested scopes; CSRF-safe; no open redirects; `skip_consent` first-party only.
- [ ] Decide DPoP: enable for high-value tools, keep DB replay store; otherwise short-lived access tokens + rotation, and treat bearer leakage (logs!) as P1 — never log `Authorization` headers or tokens.
- [ ] `client_credentials`: default-deny (`[]`), admin-assign + privilege-gate every machine scope.
- [ ] Rate-limit `/oauth2/*` and `/mcp` at the app or proxy layer with prefix-aware IPv6 bucketing; confirm CIMD governor defaults fit expected client diversity.
- [ ] Revocation story: consent delete → token revoke; user-delete session/token cleanup; rotate JWKS signing keys with `kid` rollover plan.
- [ ] Interop test against real clients (Claude/Cursor/ChatGPT) *before* launch — DCR-vs-CIMD and `legacy:"reject"` mismatches are the likeliest launch-day breakage, not security bypasses.
- [ ] Audit `@hono/node-server@1.x` (GHSA-frvp) separately — affects takibi-api directly.

## Verdict

**No security blocker to using this stack for a public MCP server — provided you ship the new packages, not the old plugin names.** The current `@better-auth/mcp@1.7.x` + `@better-auth/cimd` + SDK v2 composition has secure defaults (no implicit DCR, pinned CIMD transport with RFC 6890 rejection, resource-bound tokens, fail-closed refresh/client-credentials), no published CVE against it specifically, and takibi-api's `better-auth 1.7.5` base already clears the 1.7.0 resource-binding fix and the 1.7.3 SSO fix. Residual risk is real but manageable: a young stack on draft specs with a fast-moving 2026 advisory history (expect to track advisories closely), plus two burdens the stack explicitly leaves you — a truthful consent screen and a non-confused-deputy tool layer.
# better-auth MCP route: OAuth for all three directories?

Tentative verdict: **GO with caveats.** The better-auth MCP stack
(`@better-auth/mcp` + `@better-auth/cimd` + `jwt()` + MCP SDK v2) satisfies
the OAuth contract of ChatGPT and Claude directories as documented, has no
known CVE against the new packages, and needs no hand-rolled protocol work.
Effort is medium (~3–5 days), dominated by integration surface (consent page,
login interplay, drizzle migration, tool surface) — not protocol. Riskiest
gap: protocol-profile mismatch (`legacy: "reject"` vs clients on older
specs), which must be verified live before submission. Detail tracks:
`mcp-track-howto.md`, `mcp-track-sec.md`, `mcp-track-comply.md`.

## 1. Compliance matrix (auth only)

| Requirement | ChatGPT | Claude | Grok |
|---|---|---|---|
| OAuth 2.x per-user flow | Yes | Yes (§5D) | N/A |
| RFC 9728 discovery + 401 | Yes | Yes | N/A |
| CIMD, no DCR | Preferred | Default | N/A |
| PKCE S256 advertised | Yes | Yes | N/A |
| RFC 9207 iss | Yes | Not required | N/A |
| Resource/aud binding | Yes | Yes | N/A |
| Rotation + invalid_grant | Yes | Yes | N/A |
| JWT + JWKS | Yes | Yes | N/A |
| Loopback (Claude Code) | N/A | Partial, test live | N/A |
| Enterprise SSO | Partial (needs email) | No, opt-in only | N/A |

Per-platform verdicts: **ChatGPT listable-with-caveats** (auth fully
satisfied; remaining gates are implementer-owned: tool annotations,
securitySchemes, domain challenge, test cases, video, demo account).
**Claude listable-with-caveats** (§5D satisfied via CIMD default path;
Enterprise Managed Auth unavailable but opt-in; one live loopback test
needed). **Grok listable** (no auth requirements; catalog mechanics only).

Enterprise notes: OpenAI workspace restrictions need `email` scope enabled
(operator config). Claude EMA needs a jwt-Bearer [REDACTED] better-auth lacks —
enterprise users fall back to standard interactive OAuth, which Claude
explicitly supports.

FAIL flags (each fails submission independently of OAuth quality):
missing CIMD with DCR off; `resource` ≠ exact MCP URL (Claude);
OpenAI demo account with MFA, missing 5+3 test cases/video, wrong
annotations; Claude merged read/write tools or missing test creds;
Grok unpinned SHA or private source repo.

## 2. Security summary

Provided by the stack when configured per docs: resource-bound tokens
(RFC 8707 aud), RFC 9728 challenges + step-up 403s, no implicit DCR,
CIMD pinned transport (single DNS resolution, RFC 6890 rejection, no
redirects, 5s/5KiB caps), OAuth 2.1 baseline (S256 PKCE, exact redirect
match, iss mix-up defense, rotation), optional DPoP with DB replay store,
stateless SDK v2 transport.

Explicitly NOT provided: client trust policy (any HTTPS doc can be a
CIMD client — allowlist/audit is ours); localhost-redirect impersonation
defense (consent screen must show client + redirect host); tool-layer
safety (injection, confused deputy, no token passthrough upstream);
Bearer [REDACTED] hygiene; secrets at rest; login/consent page security
(CSRF, open redirects); deployment identity (`BETTER_AUTH_URL`, secret,
trustedOrigins, proxy headers).

CVE landscape: the June-2026 critical/high cluster (incl. 9.1
refresh-token minting, `javascript:` redirect XSS, `alg:none`) hit the
**deprecated core** `mcp`/`oidcProvider` plugins — not the new
`@better-auth/*` packages, which inherit the hardened oauth-provider
base. No published CVE against `@better-auth/mcp` or `@better-auth/cimd`
as of 2026-10-01. Adjacent: MCP SDK shared-server leak fixed in 1.26.0
(use v2 per-request factory); `@hono/node-server@1.x` advisory needs a
separate audit (repo pins ^1.14.0).

Top footguns: wrong (deprecated) `mcp` package; `resource` trailing-slash
mismatch; shared `McpServer` across requests; hand-rolled CIMD fetch
(DNS rebinding); enabling DCR "for compatibility"; consent screen hiding
context; split issuer/JWKS defaults behind a proxy.

## 3. Integration effort for takibi-base (apps/api, better-auth 1.7.5)

Steps: bump to 1.7.7 + add `@better-auth/mcp`, `@better-auth/cimd`,
`@modelcontextprotocol/server`; wire `jwt()` + `mcp()` + `cimd()` into
`auth.ts` (open: does `better-auth/minimal` suffice?); hand-merge new
OAuth/JWKS tables into drizzle schema + migration (repo doesn't use
`auth migrate`); build `/consent` page (new web surface, server-side
signed-query check); resolve hash-routed SPA login vs path-based
`loginPage` redirect; add MCP route (`createMcpHandler` factory,
POST-only, `requireMcpAuth`); plumb `/.well-known/*` discovery through
Hono; scope design + per-org tool authorization (product work); tests.

Estimate: **M (~3–5 days)**. Protocol is packaged; cost is consent page,
login interplay, drizzle merge, well-known plumbing, and the unscoped
tool surface. Open: minimal-vs-full import, canonical `resource` value,
hash-router bridging, scope→org mapping, JWT non-revocation acceptability,
`oauthClientAssertion` cleanup job.

## 4. What must be proven live before submission

- OpenAI developer-mode connection + Claude custom connector + MCP
  Inspector against the staging server (catches DCR-vs-CIMD and
  `legacy:"reject"` mismatches — the likeliest launch-day breakage).
- Do NOT ship `legacy: "reject"` for directory listings until verified
  against real directory clients (2026-07-28 defaults vs 2025-11-25
  clients is the single riskiest gap).
- Claude Code loopback callback with a typeless CIMD client.
- Consent screen shows client name, redirect host (localhost warning),
  scopes; session-cookie gated, CSRF-safe.

## Sources

better-auth [mcp](https://better-auth.com/docs/plugins/mcp),
[cimd](https://better-auth.com/docs/plugins/cimd),
[oauth-provider](https://better-auth.com/docs/plugins/oauth-provider),
[jwt](https://better-auth.com/docs/plugins/jwt),
[June-2026 security post](https://better-auth.com/blog/security-update-june-2026);
MCP [authorization](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization);
OpenAI [auth](https://developers.openai.com/plugins/build/auth),
[guidelines](https://developers.openai.com/plugins/plugin-guidelines),
[submission](https://developers.openai.com/plugins/deploy/submission);
Claude [auth](https://claude.com/docs/connectors/building/authentication),
[review-criteria](https://claude.com/docs/connectors/building/review-criteria),
[EMA](https://claude.com/docs/connectors/building/enterprise-managed-auth),
[policy §5D](https://support.claude.com/en/articles/13145358-anthropic-software-directory-policy);
[xAI marketplace](https://github.com/xai-org/plugin-marketplace).

## 5. External review addendum (Grok, then Codex — both read-only, doc-verified)

Both reviewers: **GO to integrate, NO-GO to submit until live staging
proves the connects.** Codex confirmed all six Grok corrections (details
in mcp-grok-review.md, mcp-codex-review.md):

- Corrected: CIMD 5s fetch is not Claude's endpoint budget (10s/30s are
  separate clocks); OpenAI page does not require rotation/`invalid_grant`
  (Claude rules); ChatGPT has a third identity path (predefined client),
  so "no CIMD+DCR = no path" was too absolute; 2025-11-25 is the
  authorization spec date, not proof `legacy:"reject"` breaks clients
  (keep the live test anyway); ChatGPT/Codex redirects differ by surface.
- `none` + `private_key_jwt` intersection: server must advertise and
  honor the overlap with ChatGPT's CIMD doc — inspect live metadata,
  don't assume.
- New must-haves both endorsed: every advertised scope must be grantable
  (both directories request all of them); issuer/resource/aud exact
  strings (trailing slashes fail); ChatGPT linking UI needs
  `securitySchemes` + `_meta["mcp/www_authenticate"]` on tool errors;
  admit client egress (Claude 160.79.104.0/21, OpenAI ranges + client
  cert) to the authorization server.
- Codex additions: JWT is our implementation choice, not a directory
  mandate (opaque + introspection also satisfies aud validation);
  report overstated "June CVEs hit only deprecated core" — Highs also
  hit current `@better-auth/oauth-provider` (fixed in proposed
  versions, track advisories); enterprise email needs actually-verified
  addresses, not just the scope; Grok "listable" is premature until a
  real `.mcp.json` plugin completes per-user OAuth in Grok Build.
- Refuted: 30s refresh-reuse window as a demonstrated Claude conflict
  (same rotated response is returned — still worth one live refresh
  test, not a blocker).

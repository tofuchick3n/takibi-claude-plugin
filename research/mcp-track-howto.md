Research complete. All reads were read-only; no files modified.

# better-auth MCP plugin: how it works + integration cost for takibi-base

## (1) Architecture summary

**Package roles** (each does one layer; all four are required):

| Package | Role |
|---|---|
| `better-auth` + `jwt()` (built-in plugin) | Stable signing key for access/ID tokens; serves `/jwks` so the resource server can verify tokens locally without a DB round-trip |
| `@better-auth/mcp` → `mcp()` | **Is** the OAuth provider (do NOT also register `oauthProvider()`). Configures OAuth 2.1 with MCP resource binding, serves RFC 9728 protected-resource metadata, and exports the `requireMcpAuth` route wrapper / `createMcpProtectedRequestHandler` |
| `@better-auth/cimd` → `cimd()` | Client identity without registration: validates the client's self-hosted HTTPS metadata document (client_id = document URL), persists via the OAuth provider's canonical registration path. MCP profile requires `metadataProfile: "mcp-2026-07-28"` + Node transport `fetchClientMetadataResource` from `@better-auth/cimd/node` |
| `@modelcontextprotocol/server` v2 | Owns the stateless MCP/JSON-RPC transport only (`createMcpHandler`, `McpServer`, `registerTool`). Knows nothing about auth; the `POST` handler it returns gets wrapped by `requireMcpAuth` |

**OAuth 2.1 flow (MCP 2026-07-28 profile):** client hits MCP route with no token → `401` + `WWW-Authenticate` pointing at resource metadata → fetches RFC 9728 protected-resource metadata → discovers authorization server via RFC 8414 (or OIDC discovery) → client identity via **CIMD** (DCR deprecated, never enabled implicitly) → authorization-code + S256 PKCE with RFC 8707 `resource` indicator in both authorize and token requests → resource-bound access token (`aud` = the `resource` identifier) + optional refresh via `offline_access`. `requireMcpAuth` verifies signature/issuer/audience/expiry against JWKS locally, enforces DPoP (RFC 9449) for DPoP-bound tokens, and issues RFC 6750 `insufficient_scope` 403s for step-up authorization.

**Endpoints served** (under the Better Auth base path — here `/v1/auth`, so e.g. `/v1/auth/oauth2/authorize`): `/oauth2/authorize`, `/oauth2/token`, `/oauth2/userinfo`, `/oauth2/register` (only if DCR explicitly enabled), `/oauth2/consent`, `/oauth2/introspect`, `/oauth2/revoke`, `/jwks` (from `jwt()`), plus discovery: `/.well-known/oauth-protected-resource` (+ resource-path alias), `{issuer}/.well-known/oauth-authorization-server`, and `{issuer}/.well-known/openid-configuration` when `openid` is used. Note the issuer has a base path (`/v1/auth`), so well-known URLs live at the issuer-inserted location, and the Hono forwarder must pass those URLs through to `auth.handler` (it currently forwards only `/v1/auth/*` — discovery URLs at `/.well-known/...` root will need explicit routes using the `oauthProvider*Metadata` helpers or equivalent).

## (2) Concrete integration steps for THIS repo

1. **Deps:** `npm i better-auth@^1.7.7 @better-auth/mcp @better-auth/cimd @modelcontextprotocol/server` in `apps/api` (zod already present). Repo is on better-auth **1.7.5** but `@better-auth/mcp@1.7.7` peers on `better-auth@^1.7.7` — a patch bump, likely trivial.
2. **`apps/api/src/lib/auth.ts`:** add `jwt()`, `mcp({ loginPage, consentPage, resource: "https://<prod-host>/mcp" })`, `cimd({ fetchClientMetadataResource, metadataProfile: "mcp-2026-07-28" })` to `plugins`. Open question whether `better-auth/minimal` suffices or the full `better-auth` import is needed (see §4).
3. **DB schema:** new tables `oauthClient`, `oauthAccessToken`, `oauthRefreshToken`, `oauthConsent`, `oauthClientAssertion` (+ `jwks` from jwt plugin). Repo does NOT use `npx auth migrate` — it uses drizzle-kit + `./drizzle/*.sql` applied at boot — so run `npx auth generate`-equivalent and hand-merge into `src/db/schema.ts` + a new `drizzle/0031_*.sql` migration.
4. **Login page:** `mcp()` redirects unauthenticated authorize requests to `loginPage`. Repo login is a **hash route** (`#/login`) in the SPA with a hand-rolled fetch wrapper (`apps/web/src/lib/api.ts`, no better-auth client lib). Need to verify the OAuth plugin's "new session continues the flow" handoff works with a hash-routed SPA login, or add a real `/sign-in` path (hash routes never reach the server, so the plugin's redirect + signed `oauth_query` resumption needs care).
5. **Consent page (new, required):** build a `/consent` page that verifies the signed query server-side (`verifyOAuthQueryParams`, secret stays server-side), renders client/scope/claims, and calls `POST /oauth2/consent { accept, scope?, claims? }`. Repo has no consent UI and no better-auth web client, so this is new web + thin API work.
6. **MCP route (new):** e.g. `apps/api/src/routes/mcp.ts` — fresh `McpServer` per request via `createMcpHandler(..., { legacy: "reject" })`, export POST-only, wrap with `requireMcpAuth(auth, handler, { resource })`. Mount in `app.ts`. Register actual tools (the product decision — which Takibi data/tools to expose).
7. **Discovery plumbing:** ensure `/.well-known/oauth-protected-resource*` and issuer-alias well-known routes reach `auth.handler`; add CORS `GET` allowance for local Inspector testing per docs.
8. **Tests:** repo's `verify` gate runs tsc + eslint + vitest; add coverage for the MCP route auth wrapper and the drizzle migration (incl. PGlite lane).

## (3) Effort estimate: **M (medium, ~3–5 days)**

Reasoning: the *protocol* work is genuinely packaged — no hand-rolled OAuth, JWKS, DPoP, or CIMD fetching (the `@better-auth/cimd/node` transport handles the DNS-pinning/SSRF-critical part). What makes it M, not S: (a) the consent page is a from-scratch web surface with a server-side signature check, in a codebase with no better-auth client; (b) hash-routed SPA vs. path-based `loginPage`/`consentPage` redirects needs a bridging decision; (c) drizzle schema must be hand-merged (repo doesn't use `auth migrate`); (d) the actual MCP tool surface (which tools/scopes, scope→org-membership mapping) is unscoped product work; (e) well-known routing through the Hono mount needs explicit wiring. S would only hold if login/consent already existed as paths.

## (4) Open questions

1. Does `mcp()` work with the `better-auth/minimal` import the repo uses, or must `auth.ts` switch to the full `better-auth` entry? (Docs always show the full import.)
2. What is the canonical `resource` identifier in prod (must be HTTPS, no query/fragment)? It doubles as the token `aud` and must match everywhere.
3. Hash-router compatibility: can `loginPage` be `#/login`, or do we add real `/sign-in` + `/consent` paths (server-rendered or SPA fallback entries)?
4. Scope design: which scopes (e.g. `mcp:tools`, per-resource scopes), and how do they map to orgs/memberships/roles? `requireMcpAuth` only checks scope strings; object-level auth stays in the tools.
5. JWT access tokens can't be individually revoked — acceptable with short lifetimes + session-linked `sid`, or do we need opaque tokens for some grants?
6. Multi-instance: DPoP replay store defaults to the DB adapter (fine); `oauthClientAssertion` rows need our own expiry cleanup job.

**Doc URLs cited:** [MCP plugin](https://better-auth.com/docs/plugins/mcp), [CIMD](https://better-auth.com/docs/plugins/cimd), [OAuth Provider](https://better-auth.com/docs/plugins/oauth-provider), [JWT](https://better-auth.com/docs/plugins/jwt), [MCP authorization spec 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization). Version facts from `npm view` (`@better-auth/mcp@1.7.7` peers `better-auth@^1.7.7`; `@modelcontextprotocol/server@2.2.0`).

**Verdict:** The crypto-and-protocol core is genuinely easy — better-auth + the MCP SDK absorb OAuth 2.1, PKCE, CIMD, JWKS, DPoP, and discovery, and the repo's Node/Hono/Drizzle stack matches the documented happy path with only a 1.7.5→1.7.7 bump. But it is secretly a *medium* project, not an afternoon, because the remaining work is all integration surface this repo doesn't have yet: a consent page, path-based login interplay with a hash-routed SPA, hand-merged drizzle migrations, well-known route plumbing, and — the real scope risk — deciding and building the actual MCP tool surface with correct per-org authorization. Budget for the pages and the product surface, not the protocol.
I'll read the two reports in full and spot-check the load-bearing claims against the official docs.The reports are in. Next I’ll check the load-bearing claims against the official docs.# Review of the MCP OAuth directory reports

Build the stack. Do not submit it yet. The auth matrix is mostly right. A few green cells run ahead of the docs, and several submission failures sit outside the FAIL list.

## Factual errors

**CIMD fetch timeout is the wrong clock.** Row 8 treats better-auth's 5 second CIMD fetch as evidence you meet Claude's latency budget. Claude waits 10 seconds for your discovery, registration, and token endpoints, and 30 seconds for refresh. The 5 second cap is better-auth fetching the client metadata document. https://claude.com/docs/connectors/building/authentication https://better-auth.com/docs/plugins/cimd

**Plural auth methods.** The comply report says better-auth ignores unknown plural members. The CIMD doc says unknown top-level members are ignored and never persisted. ChatGPT's production CIMD publishes `token_endpoint_auth_methods_supported` as `none` and `private_key_jwt`, plus a singular `token_endpoint_auth_method` of `private_key_jwt`. ChatGPT uses `private_key_jwt` when your authorization server also advertises it, and otherwise uses another method in the intersection. A server that stores only the singular method, and advertises only `none`, will disagree with ChatGPT about the token request. https://developers.openai.com/plugins/build/auth https://better-auth.com/docs/plugins/cimd

**OpenAI rotation and `invalid_grant`.** OpenAI says access and refresh tokens may expire or rotate, and a deleted client credential surfaces as `invalid_client`. That page does not require refresh rotation or `invalid_grant`. Public-client rotation, `invalid_grant`, and form-urlencoded token bodies are Claude rules. https://developers.openai.com/plugins/build/auth https://claude.com/docs/connectors/building/authentication

**ChatGPT still has a third identity path.** FAIL flag 1 says that with CIMD absent and DCR off, ChatGPT has no client identity path. The auth guide lists a predefined OAuth client beside CIMD and DCR. The flag still holds for a directory install that expects CIMD or DCR. It is too absolute as written. https://developers.openai.com/plugins/build/auth

**2025-11-25 is the authorization spec.** The riskiest-gap paragraph reads directory clients as 2025-11-25 clients that `legacy: "reject"` will drop. OpenAI points that date at the authorization spec. `legacy: "reject"` is the setting better-auth tells you to pass so the SDK v2 route rejects the session-oriented 2025 transport and pins negotiation to 2026-07-28. Keep the live test. The auth pages do not already prove the transport mismatch. https://developers.openai.com/plugins/build/auth https://better-auth.com/docs/plugins/mcp https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization

**Redirect allowlist.** Row 11 says ChatGPT uses https://chatgpt.com callbacks only. The guide also says to copy the production redirect from the MCP server management page, and that callback details differ by surface. Published plugins run in ChatGPT and Codex. https://developers.openai.com/plugins/build/auth

## Missing requirements that can fail a submission

**Scopes you advertise, you must issue.** ChatGPT requests every OIDC scope in authorization-server `scopes_supported`, and that scope has to be enabled on the CIMD, manual, or DCR client. Claude, with no `scope` on the 401, requests every protected-resource `scopes_supported` value, and adds `offline_access` when the authorization server lists it. Enterprise email is the special case already in the matrix. The general case can fail a normal connect. https://developers.openai.com/plugins/build/auth https://claude.com/docs/connectors/building/authentication

**Issuer bytes and well-known routing.** ChatGPT and Codex compare `issuer`, protected-resource `authorization_servers`, and the `iss` response as exact strings. Trailing slashes fail that compare. With a base path, better-auth serves discovery at the issuer-inserted well-known URL. This repo's Hono mount forwards `/v1/auth/*` only, so a 404 there fails both directories before a token is issued. https://developers.openai.com/plugins/build/auth https://better-auth.com/docs/plugins/mcp

**Audience on ChatGPT.** `resource` mismatch is filed as a Claude-only FAIL. ChatGPT sends the protected-resource `resource` value on authorize and token requests and expects that string as the token audience. Path or slash drift fails ChatGPT as well. https://developers.openai.com/plugins/build/auth

**Linking UI.** ChatGPT shows the OAuth linking UI only when the tool declares `securitySchemes` and the tool error carries `_meta["mcp/www_authenticate"]` with `error` and `error_description`. The verdict calls this implementer-owned. For any authenticated tool, it is a connect failure. https://developers.openai.com/plugins/build/auth

**Egress.** Claude calls the authorization server from `160.79.104.0/21`, the same range as MCP calls. A WAF in front of the identity provider drops discovery while the MCP host still answers. ChatGPT documents published egress ranges and presents a client certificate. Blocking either fails review. https://claude.com/docs/connectors/building/authentication https://developers.openai.com/plugins/build/auth

**Claude's CIMD gate is two fields.** Claude selects CIMD only when authorization-server metadata has `client_id_metadata_document_supported: true` and `none` inside `token_endpoint_auth_methods_supported`. Otherwise it falls back to DCR. The public MCP page confirms the first flag when `cimd()` is installed. It does not show the `none` advertisement. That claim points at GitHub `metadata.ts`, which I did not re-fetch. If the advertisement is missing and DCR is off, Claude directory auth fails. https://claude.com/docs/connectors/building/authentication https://better-auth.com/docs/plugins/mcp

**Refresh grace.** Claude wants the new refresh token in the response that invalidates the old one. `mcp()` defaults `refreshTokenReuseInterval` to 30 seconds and will accept the previous refresh token inside that window. Prove Claude tolerates the window before submission. https://claude.com/docs/connectors/building/authentication https://better-auth.com/docs/plugins/mcp

## Verdicts

Claude is listable with caveats. Enterprise Managed Auth stays opt-in, and interactive OAuth remains the documented path. The typeless Claude Code loopback test is still open. The `none` advertisement and the scope list belong at the same weight as that test.

ChatGPT is listable with caveats too, and "auth fully satisfied" overshoots. Rows 1, 2, 3, 5, 6, 7, and 9 hold. Rows 4 and 8 are ahead of the page. Linking UI, advertised scopes, the issuer string, and the Codex redirect are handshake blockers, in the same class as the submission package.

I did not re-fetch the xAI marketplace. Nothing on the five auth pages contradicts "no auth requirements, catalog mechanics only."

## Go / no-go

GO to integrate `@better-auth/mcp`, `@better-auth/cimd`, and `jwt()` on better-auth 1.7.7. The packaged pieces I checked are real. RFC 9728 metadata, no implicit DCR, S256 in the documented profile, resource-bound tokens, JWKS verification on `requireMcpAuth`, the 401 challenge, and the pinned CIMD transport.

NO-GO to submit to either OAuth directory until staging shows three connects. ChatGPT developer mode, accepting whichever of `none` and `private_key_jwt` ChatGPT sends. A Claude custom connector whose `resource` equals the URL you type, with `none` visible in authorization-server metadata, and with scopes limited to ones the server grants. Claude Code loopback on a random port. Leave `legacy` unset to `reject` until one of those clients finishes a tool call. Confirm Anthropic and OpenAI egress can reach the authorization server.

Checked better-auth MCP, better-auth CIMD, OpenAI plugin auth, Claude connector authentication, MCP authorization 2025-11-25, and MCP authorization 2026-07-28. I did not re-open the OAuth provider page, the directory policy, the submission guidelines, or the xAI repo.

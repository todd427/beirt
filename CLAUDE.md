# Beirt — notes for agents

Beirt is a bring-your-own-key fork of [Duel](https://github.com/todd427/duel).
One file, no build, no backend. Read this before changing anything.

## The invariant

**There is no server.** The user supplies their own Anthropic API key and the
page calls `api.anthropic.com` directly. Do not reintroduce a proxy, a Pages
Function, a serverless route, or a "small backend just for X". If a feature
seems to need one, it does not belong in this fork — it belongs upstream in
Duel, which already has that architecture.

Corollaries:

- No `functions/`, no `api/`, no `wrangler.toml`, no `.env`.
- No secrets in the repo. The key comes from the user at runtime.
- No build step. No npm, no bundler, no framework, no TypeScript.
- Everything lives in `index.html`. Resist splitting it up; a single file is a
  feature here, because the deploy story is "open the file".

## Structure of index.html

1. `<style>` — theme blocks first (`[data-theme="paper|folio|slate|ink|terminal"]`),
   then reset, then components in the order they appear in the markup.
2. Markup — welcome overlay, settings dialog, then the app (header, panes,
   input area).
3. `<script>` — constants, state, key storage, verification, welcome, settings,
   MCP servers, theme, then the original Duel functions largely unchanged.

## Themes

Themes are a semantic token layer. Every component reads `var(--bg)`,
`var(--surface)`, `var(--text)`, `var(--left-*)`, `var(--right-*)` and friends —
never a literal colour. Adding a palette means adding one `[data-theme]` block
and one entry to the `THEMES` array. Do not hard-code a hex outside a theme
block; the `.theme-btn[data-t=…]` swatches are the one deliberate exception,
because a swatch must show its own colour regardless of the active theme.

`auto` is a stored preference, not a palette: it resolves to Paper or Folio from
`prefers-color-scheme`.

Terminal is carried over from [rialú](https://rialu.ie) and is the only theme
that changes the typeface as well as the colours.

## Key handling

- Session mode → `sessionStorage.beirt_key`. Remember mode → `localStorage.beirt_key`.
- `getKey()` is the only reader. `storeKey()` is the only writer.
- Never log the key, never put it in a URL, never send it anywhere but the
  Anthropic API.
- Every request path must handle 401/403 by setting `err.keyProblem = true` so
  the error bubble offers a route back into settings.

## MCP

- Deny-by-default is the default, not the only option: `default_config:
  { enabled: false }` plus one explicit entry per permitted tool. The `*`
  sentinel in a server's `tools` means the whole toolset — it sends
  `default_config: { enabled: true }` and no per-tool entries. It is a
  deliberate, visible choice, never the state a new server starts in.
- A server cannot be enabled with an empty allow-list. Keep that guard: the
  empty row must say what it means, since a server that looks configured but
  is silently unsent is indistinguishable from one that is down.
- Allow-list entries are the server's own tool names, unprefixed. The model
  sees them namespaced (`mnemos_query_memory`), but `configs` is keyed by
  what the server exposes (`query_memory`). Verified working against a live
  server; do not "correct" the allow-list to the prefixed form.
- Tokens follow the same keep mode as the API key.
- Exported JSON strips tokens. Keep it that way.
- Do not hard-code any server. The upstream project's private servers were
  removed on purpose; a stranger's key cannot reach them and shipping them would
  send dead configuration on every request.
- Nothing MCP is sent in a féirín session, and the ways in are hidden. An
  `mcp_servers` block carries the user's own bearer tokens, and the server list
  lives in `localStorage` — so on a shared machine a claimant would otherwise
  push the previous user's secrets through Féirín's proxy.

## Féirín

A féirín is a QR gift voucher. `#feirin=<token>` in the fragment puts the page
into a voucher session: requests go to `https://feirin.foxxelabs.ie/v1/messages`
with `Authorization: Bearer`, and Féirín holds the key and counts what is left.

This does not break the invariant above, and must not be allowed to. It is a
session the *user* opts into by scanning someone's voucher, gated on one
condition, and a session without a token behaves exactly as it always did:
`api.anthropic.com`, own key, own spend meter, all eight models, all tools,
nothing in the path. Do not hoist `FEIRIN_URL` and `API_URL` into one constant
— Féirín being down must stay invisible to a BYOK user. Do not add a proxy for
anything else.

Three things that are easy to get wrong:

- **`X-Feirin-Comparison`.** One unit is one two-pane comparison, but the proxy
  sees two independent POSTs. Both panes of a question send the same id and the
  second is free; a missing header is charged every time. Forgetting it fails
  silently and quietly halves every voucher, so `send()` mints one id for the
  question and hands it to both panes. A cross-send is its own question and
  gets its own id.
- **The gauge is units, not money.** `X-Feirin-Units-Remaining` is the only
  authority, it arrives on refusals too, and it is already decremented — both
  panes report the same number, so do not subtract. `PRICES`, `recordUsage`
  and the spend meter are for BYOK sessions, where Beirt really is watching its
  own spend. Never put a currency figure on a féirín.
- **The proxy serves two models, clamps `max_tokens` to 2048, and allows six
  calls a minute with two in flight.** `applyFeirinUI()` matches the UI to
  that rather than letting a claimant find out by 400, and hides the Web and
  Code toggles — server-side search bills on top of what a comparison was
  budgeted at, and the gift-giver wears the difference. MCP is hidden with
  them, for the token rather than the cost; see the MCP section above.

The copy that promises nothing is proxied and nothing is logged is false in a
voucher session and is swapped there. Féirín logs which model was called and
how many tokens moved — not prompts, not replies. Keep that honest.

## Naming

All persisted keys are prefixed `beirt_`. If you add one, follow the prefix:
`beirt_key`, `beirt_keep`, `beirt_theme`, `beirt_fs`, `beirt_spend`,
`beirt_web`, `beirt_settings`, `beirt_mcp`, `beirt_mcp_tokens`, `beirt_mt`,
`beirt_nostream`, `beirt_cur`, `beirt_code`, `beirt_feirin`,
`beirt_feirin_units`.

The last two are `sessionStorage` only, deliberately. Everything else follows
the user's keep mode.

## Style

Match the existing voice: terse comments that say *why*, not *what*; section
rules as `// ── Name ────`; no comment that restates the line below it. Copy in
the UI is plain and matter-of-fact — no exclamation marks, no emoji beyond the
handful already carried over from Duel's chrome.

## Credit

FoxxeLabs credit appears in three places: the first-run footer, Settings →
About, and the README. Keep all three if you touch that area.

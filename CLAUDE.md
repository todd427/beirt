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

- Deny-by-default: `default_config: { enabled: false }` plus one explicit entry
  per permitted tool. Never send a toolset with everything enabled.
- A server cannot be enabled with an empty allow-list. Keep that guard.
- Tokens follow the same keep mode as the API key.
- Exported JSON strips tokens. Keep it that way.
- Do not hard-code any server. The upstream project's private servers were
  removed on purpose; a stranger's key cannot reach them and shipping them would
  send dead configuration on every request.

## Naming

All persisted keys are prefixed `beirt_`. If you add one, follow the prefix:
`beirt_key`, `beirt_keep`, `beirt_theme`, `beirt_fs`, `beirt_spend`,
`beirt_web`, `beirt_settings`, `beirt_mcp`, `beirt_mcp_tokens`.

## Style

Match the existing voice: terse comments that say *why*, not *what*; section
rules as `// ── Name ────`; no comment that restates the line below it. Copy in
the UI is plain and matter-of-fact — no exclamation marks, no emoji beyond the
handful already carried over from Duel's chrome.

## Credit

FoxxeLabs credit appears in three places: the first-run footer, Settings →
About, and the README. Keep all three if you touch that area.

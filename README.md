# Beirt

**Two independent Claude sessions. One problem. No shared context.**

Beirt is a single-file browser tool for running two Claude API sessions side by side. Send the same question to both simultaneously, watch them diverge, then fire one's answer at the other for reaction.

You bring your own Anthropic API key. There is no server, no account, and nothing to deploy.

Run it at **[beirt.foxxelabs.ie](https://beirt.foxxelabs.ie)** or open `index.html` from a
local clone. They are the same file, and neither one has anything behind it.

*Beirt* (/bʲɛrˠtʲ/ — "bert", with the final t softened toward *tch*) is Irish for a pair of people.

---

## What it does

- Two independent panes — **α Alpha** and **β Beta** — each with its own model, system prompt, and conversation history
- **Streaming** — responses render token-by-token live in each pane (both panes stream in parallel)
- **Reasoning** — on models that think by default (Fable 5, Opus 5, Sonnet 5) the pane streams a summary of the model's reasoning while it works, then folds it away once the answer starts. Thinking is billed the same whether or not it is shown, so this costs nothing extra
- **Web access** — optional `🌐 Web` toggle gives both panes Anthropic-hosted web search + fetch; tool activity is shown inline
- **MCP connections** — add your own MCP servers from **⚙ Advanced** in the header: URL, bearer token, and a per-server allow-list of permitted tools. Off by default, deny-by-default, remembered between sessions, exportable as JSON
- **Route selector** — send to Alpha only, Both (parallel), or Beta only
- **Cross-send** — send any response from one pane to the other as a new user message
- **Presets** — one-click system-prompt pairs (Sceptic ⚔ Builder, Red ⚔ Blue team, Line ⚔ Dev editor)
- **Configurable max tokens** — set the per-reply output budget; truncated replies are flagged
- **File attachments** — images (vision), PDFs, text/code files; drag and drop works
- **Spend tracking** — a running session cost estimate + token totals, computed per-model from the API's `usage`
- **Export MD** — downloads both conversation threads as a timestamped Markdown file
- **Readable type** — message/code/compose text auto-scales with window width, plus an `A− / A+` nudge (persists)
- **Five themes + Auto** — Paper and Folio (the house pair), Slate and Ink (neutral), Terminal (monochrome green, from [rialú](https://rialu.ie)). Auto follows your OS light/dark setting
- **Persistence** — system prompts, model choices, max tokens, theme, MCP servers and toggles survive a refresh
- Enter sends. Shift+Enter is a new line.

## What it's for

Architecture review. Two system prompts:

> **α:** *You are a sceptical architect. Challenge every assumption. Find the failure mode.*
> **β:** *You are a pragmatic builder. Find the path forward.*

Send the same design problem to both. Cross-send β's proposal to α. Watch the sceptic work on it.

Also useful for: writing feedback, argument stress-testing, comparing reasoning styles across models (Opus vs Sonnet, etc.), or just watching two independent minds work through the same hard question simultaneously.

## Requirements

- **An Anthropic API key.** Get one at [console.anthropic.com/settings/keys](https://console.anthropic.com/settings/keys). Usage is billed to your own account — the spend counter in the header is an estimate, your console is the source of truth. **Use a dedicated key with a spend limit.**
- A modern browser.

That is all. There is no build step, no npm, no server, and no hosting requirement.

## Usage

```bash
git clone https://github.com/todd427/beirt
cd beirt
open index.html          # or just double-click it
```

Paste your key into the first-run screen, choose whether to keep it for the session or remember it on the device, and start.

To serve it instead of opening the file directly, any static host will do — Cloudflare Pages, GitHub Pages, `python -m http.server`. Nothing on the server side is required, because there is no server side. The hosted copy is this repo's `main` branch on Cloudflare
Pages, deployed straight from Git with an empty build command and no `wrangler.toml`.

## Where your key goes

Requests go from the page straight to `api.anthropic.com` using the API's opt-in direct-browser-access header. Nothing is proxied, nothing is logged, and no third party sees the key.

That does mean the key is present in the page, which is a real trade — see [SECURITY.md](SECURITY.md) before you choose "remember on this device".

## MCP connections

Beirt is just another client for the Messages API, so it can reach any MCP server you can. Settings → MCP takes a name, a URL, a bearer token, and an explicit list of allowed tool names. A server starts disabled with an empty allow-list and cannot be enabled until you name at least one tool.

Connections configured in other Claude products cannot be imported. Those are configuration of *those products*, held against your account there; an API key authenticates model access and billing, and no endpoint hands out an app's saved connector list. Every API client declares its own `mcp_servers` per request. Beirt remembers yours locally and exports them as JSON so you can move them between browsers by hand.

Exported JSON deliberately omits tokens.

## Technical notes

- Single file (`index.html`) — no build step, no npm, no framework, no backend
- Responses stream over SSE (`stream: true`), parsed from the `fetch` body reader — no SDK
- Web access uses Anthropic's server-side `web_search` + `web_fetch` tools (they run on Anthropic's infrastructure, not in the browser)
- MCP uses the Messages API's MCP connector, which rides a beta header — the most likely part of this to need maintenance
- File attachments: images → base64 vision blocks, PDFs → document blocks, text/code → fenced code blocks prepended to message
- Themes are a semantic token layer (`--bg`, `--surface`, `--text`, `--left-*`, `--right-*`, …); adding a palette means adding one `[data-theme]` block
- No analytics, no logging, no telemetry

## Models

`claude-fable-5` · `claude-opus-5` (default) · `claude-opus-4-8` · `claude-opus-4-7` · `claude-opus-4-6` · `claude-sonnet-5` · `claude-sonnet-4-6` · `claude-haiku-4-5`

Fable 5 is the most capable and the most expensive — $10/$50 per million tokens against Opus 5's $5/$25. It also requires 30-day data retention, so an org configured for zero retention gets a 400 rather than a reply.

Each pane selects independently. Which models your key can actually reach depends on your account; an unavailable model returns an API error into the pane rather than failing silently.

## Relationship to Duel

Beirt is an open, bring-your-own-key fork of [todd427/duel](https://github.com/todd427/duel). Duel routes through a Cloudflare Pages Function that holds the Anthropic key and two private MCP servers server-side, behind Cloudflare Access. That design is right for a hosted tool and wrong for one you run yourself, so this fork removes the function entirely and hands the key and the MCP configuration to the user.

The two are separate repositories rather than branches: the whole app is one file, so a branch would put the hosted and self-hosted versions in permanent conflict in the file every change touches.

## License

MIT. See [LICENSE](LICENSE).

---

Designed and built by [FoxxeLabs](https://foxxelabs.ie)

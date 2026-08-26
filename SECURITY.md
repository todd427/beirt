# Security

Beirt holds your Anthropic API key in your browser and sends it directly to
`api.anthropic.com`. That is the whole point of the fork, and it is a real
trade-off. Here is the honest version.

## The key is in the page

Anything running in the page can read it. There is no server to hide it behind
— that is what was removed. Two consequences:

- **Session-only is the default.** The key lives in `sessionStorage` and is gone
  when you close the tab. Nothing secret is written to disk.
- **"Remember on this device" writes to `localStorage`.** Convenient, and
  readable by anything with script access to the same origin. Choose it
  deliberately, not by habit.

## Use a dedicated key with a spend limit

Create a separate key for Beirt in the
[Anthropic Console](https://console.anthropic.com/settings/keys) and cap it.
If the key leaks, you revoke one key and lose nothing else. The spend counter in
the header is a local estimate to help you notice runaway usage early; it is not
a billing record.

## Third-party origins

The page loads two font families from `fonts.googleapis.com`. That is the only
external origin it touches besides the Anthropic API.

If you want a build with no third-party requests at all, self-host the fonts:
download IBM Plex Sans, IBM Plex Mono and Playfair Display, drop them next to
`index.html`, and replace the `<link>` tags with local `@font-face` rules. The
app needs no other change.

## Direct browser access

Requests set `anthropic-dangerous-direct-browser-access: true`. This is a
supported, opt-in path — the header is named the way it is because sending a
provider key from a browser is exactly the thing most apps should not do. Beirt
does it because the alternative is a server that holds your key, which is the
design this fork exists to remove. If you would rather your key never reach the
browser, run [Duel](https://github.com/todd427/duel) with your own
`ANTHROPIC_API_KEY` on Cloudflare Pages instead. That is a legitimate choice and
the upstream project supports it.

## MCP tokens

Bearer tokens for MCP servers follow the same session-or-remember choice as the
API key, so a session-only user writes no MCP secrets to disk either.

Servers are deny-by-default: a new server is disabled with an empty allow-list
and cannot be enabled until you name at least one permitted tool. Only the tools
you name are sent as enabled; everything else on that server is explicitly
disabled per request.

Exported JSON omits tokens by design, so an exported config is safe to move
between machines or paste into a chat.

## What Beirt does not do

- No analytics, telemetry, or error reporting
- No logging of prompts or responses anywhere
- No network requests except: the Anthropic API, the MCP servers you add, and
  Google Fonts
- No cookies

## Reporting

Open an issue at
[github.com/todd427/beirt/issues](https://github.com/todd427/beirt/issues).
Please do not include an API key in a bug report.

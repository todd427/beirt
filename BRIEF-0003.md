# <span style="color:#268bd2">Beirt — CC Brief 0003 · Taking a Féirín Token</span>

**Repo:** todd427/beirt · **Target:** beirt.foxxelabs.ie (static, serverless) · **Status:** brief, unbuilt
**Date:** 2026-09-05 · **Relation:** replaces the `#key=` handover that Féirín Brief 0001 §5 specified and that `claimFeirin()` implements today. Nothing outside voucher sessions changes.

## <span style="color:#268bd2">0. Why 0003 exists</span>

Féirín stopped handing out Anthropic keys. Brief 0002 turned a voucher into an
*entitlement* — "20 comparisons" counted in Féirín's own database — and put
Féirín in the serving path. Féirín's half is built, deployed and proven live.

Beirt is still built for the old bargain. `claimFeirin()` reads
`#key=sk-ant-…&budget=5&budget_usd=5.40`, stashes a euro budget in
`localStorage`, and draws the gauge by pricing tokens against `PRICES`. Every
line of that assumes Beirt holds a real key and can see what it spends.

Under 0002 it holds a féirín token worth a count of comparisons, and the only
authority on how many remain is a response header.

## <span style="color:#268bd2">1. The whole change in one paragraph</span>

When the fragment carries `feirin=<token>`, Beirt sends its Messages API
requests to `https://feirin.foxxelabs.ie/v1/messages` with
`Authorization: Bearer <token>` instead of `api.anthropic.com` with
`x-api-key`, marks both panes of a question with a shared comparison id, and
renders the gauge from `X-Feirin-Units-Remaining`. Everything else — BYOK
sessions, stances, routes, tools, the two-pane UI — is untouched.

## <span style="color:#268bd2">2. The claim handler</span>

Replace the `key=` branch of `claimFeirin()` (currently ~line 1599). The old
one stays only if you still want 0001 vouchers to work; Féirín no longer mints
them, so deleting it is cleaner.

```js
// #feirin=fv_XK7Q2.9c1f…  — one token, no budget, no key.
const p = new URLSearchParams(location.hash.slice(1));
const tok = p.get('feirin');
if (tok) {
  feirin = { token: tok };
  sessionStorage.setItem('beirt_feirin', tok);
  history.replaceState(null, '', location.pathname + location.search);
}
```

<span style="color:#2aa198">**`sessionStorage`, not `localStorage`.**</span>
A room féirín is one QR on a slide passed round a cohort, often on a shared or
borrowed machine. The old code put the key in `localStorage`, where it outlived
the session and the room. A token that dies with the tab is the right default;
`localStorage` is for the BYOK user's own key and stays as it is.

Keep `history.replaceState`. The fragment must not survive in the address bar,
the same reason Féirín puts it in a fragment rather than a query string.

## <span style="color:#268bd2">3. The comparison contract</span>

<span style="color:#dc322f">**This is the part that is not in Féirín Brief
0002, and the part that silently halves every voucher if it is missed.**</span>

A unit is one **two-pane comparison**. The proxy sees two separate
`POST /v1/messages` calls and cannot tell which two belong together. So Beirt
must say so:

```
X-Feirin-Comparison: <opaque id, ≤64 chars>
```

Both panes of one question send the **same** id; the next question uses a new
one. A `crypto.randomUUID()` per send is fine.

- first call with a new id — costs one unit
- second call with the same id — free
- third call with the same id — charged again
- **no header at all — charged every time**

The server caps a pairing at two calls inside three minutes, so a replayed id
buys nothing. Omitting the header is always the expensive choice, never the
cheap one — that is the property that makes a client-supplied discount safe,
and it is why forgetting it does not fail loudly. It just quietly turns a
20-comparison voucher into a 10-comparison one.

Route matters: `route = 'both'` sends two calls and they share one id. A
question sent to a single pane is one call and still costs one unit — a
comparison the claimant chose not to make is still a comparison spent.

## <span style="color:#268bd2">4. The gauge is units, not euros</span>

Delete the client-side cost arithmetic for féirín sessions. `PRICES`,
`spend.usd`, `budget.usd`, the EUR/USD rate and the `€1.37 / €5.00` rendering
were all estimates of someone else's meter. They have no authority now, and
0002's economics section is explicit that the euro figure on the card is
marketing rather than a cap.

Every response from Féirín — successes and refusals alike — carries:

```
X-Feirin-Units-Remaining: 17
X-Feirin-Units-Total: 20
```

Read them and render `17 of 20 comparisons left`. Keep the colour shift near
empty; drop the currency symbol entirely.

<span style="color:#2aa198">**On a streamed reply the header is sent before the
first token.**</span> Headers cannot be revised once the body is flowing, so
Féirín sends what the comparison *leaves behind* rather than what it started
with. Both panes of one comparison therefore report the same number, and that
number is already decremented. Do not subtract again.

`PRICES` and the spend meter stay exactly as they are for BYOK sessions, where
Beirt really is watching its own spend.

## <span style="color:#268bd2">5. What the proxy will and will not serve</span>

Enforced server-side, so the UI should match rather than discover them by 400.

- <span style="color:#2aa198">**Models:**</span> `claude-haiku-4-5` and
  `claude-sonnet-5` only. In a féirín session, reduce both dropdowns
  (~lines 1464 and 1486) to those two and default the panes to one each — that
  pairing *is* the demo. Anything else is refused, deliberately not
  substituted: quietly answering as a different model would make a
  model-comparison tool lie about which model said what.
- <span style="color:#2aa198">**`max_tokens`:**</span> clamped to 2048.
  The control defaults to 32000 and allows 64000, so a claimant will otherwise
  set a number that silently does nothing. Cap the input at 2048 for the
  session.
- <span style="color:#2aa198">**Rate:**</span> 6 calls/minute, 2 in flight, per
  voucher. Sending Both is exactly at the concurrency cap, which is fine; a
  retry storm is not. `MAX_RETRIES = 2` on stream faults can put a third call
  in flight — hold retries to one, or serialise them.
- <span style="color:#dc322f">**Server tools:**</span> hide the Web and Code
  toggles in a féirín session. Féirín's economics budget a comparison at
  roughly 2k in / 1k out per pane; web search bills $0.01 per search on top and
  both panes search independently, so one question with search on can cost
  several times what the unit assumed. The entitlement still holds — the
  claimant cannot exceed their comparisons — but Todd wears the difference.

## <span style="color:#268bd2">6. Errors</span>

Féirín answers in Anthropic's own error envelope, so `showError()` keeps
working. What changes is which status means what:

| status | meaning | copy |
|---|---|---|
| `401` | token missing, forged, or not a féirín | treat as `keyProblem` — the féirín is not valid |
| `403` | spent, killed, or exhausted | `budgetExhausted` — the existing "this féirín is spent" path |
| `429` | rate limit or concurrency | slow down; retry after a beat. Not exhaustion — do not say spent |
| `400` | model or `max_tokens` rejected | a UI bug, not the claimant's fault |
| `502` / `503` | Féirín cannot reach Anthropic, or is resting | "not your féirín — try again shortly" |

The existing exhaustion copy at ~line 2559 is right and should be kept; only
its trigger moves, from an Anthropic spend-limit message to a `403`.

<span style="color:#2aa198">A refusal still carries the units headers</span>
(except `401`, where there is no voucher to report on), so the gauge stays
honest through an error.

## <span style="color:#268bd2">7. Traps</span>

- <span style="color:#dc322f">**Dated model ids come back.**</span> A request
  for `claude-haiku-4-5` returns `"model": "claude-haiku-4-5-20251001"`. The
  undated form is correct to *send*; do not compare the response's `model`
  against what you asked for, and do not feed it back into the dropdown.
- <span style="color:#dc322f">**The gauge is not a wallet.**</span> Units are
  comparisons, not money. Do not reintroduce a currency figure from the
  `denomination` on the card — Féirín treats that as display-only, and a
  claimant told they have "€3.40 left" will do arithmetic that is not true.
- <span style="color:#dc322f">**No `/v1/models` on Féirín.**</span> Only
  `/v1/messages` is proxied. `MODELS_URL` and `FILES_URL` will 404. Do not
  probe for a model list in a féirín session; the two allowed models are
  known ahead of time.
- <span style="color:#dc322f">**Féirín downtime is now real downtime.**</span>
  For voucher sessions only. BYOK users must remain completely unaffected by
  it — that separation is the point, and it is easy to lose by hoisting the
  base URL into a shared constant.
- <span style="color:#dc322f">**A leaked token is bounded but live.**</span>
  It is worth `units_remaining` comparisons on two allow-listed models under a
  2048-token ceiling. `sessionStorage` and `replaceState` keep it out of the
  places it would otherwise linger.

## <span style="color:#268bd2">8. What must not change</span>

Beirt stays serverless and stays a BYOK tool. Everything in this brief lives
behind one condition — a féirín token is present — and a session without one
must behave exactly as it does today: straight to `api.anthropic.com`, own key,
own spend meter, all eight models, all tools, no Féirín in the path.

The two copy lines that promise this (~1237 and ~1304: *"Requests go straight
from this page to `api.anthropic.com`… nothing is proxied and nothing is
logged"*) become **false in a féirín session** and must be swapped for honest
wording there. Féirín logs which model was called and how many tokens moved —
not prompts, not replies — and the privacy notice on the claim page already
says so. Saying "nothing is logged" to someone whose traffic is being counted
is the one thing here that would actually be a lie.

## <span style="color:#268bd2">9. Out of scope</span>

Personal vouchers (Féirín 0002 §4 takes those off Beirt entirely — they become
a submit page and an emailed batch result); top-ups; any Féirín endpoint other
than `/v1/messages`; showing the claimant their own usage history.

## <span style="color:#268bd2">10. Build order</span>

1. Claim handler: parse `#feirin=`, `sessionStorage`, `replaceState`.
2. Transport: base URL and `Authorization: Bearer` behind the féirín flag.
3. **Comparison id on both panes.** Nothing else is worth shipping without it.
4. Gauge from headers; retire the cost estimate for féirín sessions.
5. UI constraints: two models, 2048 ceiling, tools hidden, honest copy.
6. Error mapping.

Steps 1–3 make a féirín work correctly. Steps 4–6 make it *feel* right and
keep it honest.

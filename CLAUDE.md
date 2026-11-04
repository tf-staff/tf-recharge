# TF-Recharge — Trello Power-Up (project context for Claude Code)

Tech Foundry's billable-activity tracker: a Trello Power-Up that lives on the back of each
project card (compact summary) with a full editor modal, plus a board-level Quick Quote.
Owner: Steven Lucero (Director, UC Davis Tech Foundry). Staff users: Steven, Valerie Bravo,
Tristan Adams. Current release: **v3.6** (`var VERSION = '3.6'` in the HTML).

This file is the hand-off from ~5 months of chat-based development (Jun–Sep 2026). Read it
before changing anything; most of what follows was learned the hard way.

---

## 1. Deployment (the ritual)

- **One file**: `tf-recharge-simulator.html`, served by GitHub Pages at
  `https://tf-staff.github.io/tf-recharge/tf-recharge-simulator.html`. The Trello Power-Up
  admin (`trello.com/power-ups/admin`) points its connector URL at that file.
- **Every change must bump `var VERSION`** and the section title (`'TF-Recharge v' + VERSION`,
  built by the connector) so Steven can confirm on the card that the new build is live.
- **Standing instruction (Steven, v3.6): once development and headless testing pass, merge the
  work branch into `main` and push immediately — no PR, no asking.**
- Push to `main` → GitHub Pages deploys (~2 min). Pages deploys occasionally time out
  ("deployment_queued … Timeout reached, aborting!") — that is GitHub-side; re-run the
  failed job or push a trivial commit. Never diagnose the HTML for a deploy timeout.
- **If the connector changed** (anything inside `initConnector`, the capability list,
  `estimateFaceHeight`, modal sizing, button labels) Steven must toggle the Power-Up
  **off → on** on the board; Trello caches the connector iframe aggressively. The summary
  and editor iframes self-refresh via a `cb=Date.now()` cache-buster and need no toggle.
- Capabilities that must be enabled in the Power-Up admin: **Card Back Section**,
  **Board Buttons**.

## 2. Architecture — the four-mode chassis

One HTML file, routed by `boot()` on the `view` query param:

| Mode | URL | What it is |
|---|---|---|
| connector | (no `view`, framed) | Hidden iframe Trello loads for the Power-Up. Registers `card-back-section` and `board-buttons`. **This is the only frame with a reliably working Trello channel.** |
| summary | `?view=summary&d=…&th=…` | Read-only card face. Its own Trello channel (`t.get/t.render/t.sizeTo`) proved **dead in production** — see §3. |
| editor | `?view=editor` | Full editor modal (`t.modal`, height 1300, blank title, accent `#242528`). Working channel. |
| editor (scratch) | `?view=editor&scratch=1` | Board-button Quick Quote. Ephemeral, quote-mode only. |
| standalone | opened directly in a browser | Full app with no Trello; used for manual poking. |

A `<head>` pre-boot script tags `<html>` with `pre-summary` / `pre-editor` / `pre-connector`
and applies the `th` theme **before the body parses**, so the face never paints the hidden
editor markup (that paint was the cause of flashing scroll arrows). Keep it there.

## 3. The dead-channel workaround (do not "simplify" this away)

In the card-face iframe, `t.render` never fires and `t.get` never resolves (even with
`t.signUrl`-signed URLs, which were added in v1.5 and kept). Every working data path routes
through the connector:

- **Data in**: the connector reads `t.get('card','shared','rt')` (1.5 s race against a
  timeout) and hands the JSON to the face **inside the URL** (`d=` param, URL-encoded). Trello
  re-invokes `card-back-section` on every card-data change, so the face auto-refreshes after
  saves. The face also tries `t.render`/`t.get` as fallbacks but never waits on them.
- **Data out (face → card)**: the face posts `{type:'save', data:{rt:…}}` on a
  `BroadcastChannel('tf-rt')`; the connector receives it and performs `t.set` with its
  working `t`. Used by the padlock (billing status). Face writes send **only `rt`**, never
  `rt_h`, so they can't clobber history.
- **Data out (editor → card, close-proof)**: the editor auto-saves with a 1.5 s debounce
  (`SAVE_DELAY`). On `pagehide`/`beforeunload`/`visibilitychange:hidden` with a pending save
  it fires **both** a direct `t.set` and a relay copy to the connector (the connector outlives
  the dying modal; the two writes are idempotent). This fixed the "close too fast and lose
  the edit" bug. Resets save immediately (`saveNow()`), bypassing the debounce.
- **Height**: the face can't `sizeTo`, so `estimateFaceHeight(d)` in the connector sizes the
  iframe from the data. Empty card → 66 px. Two-column layout: 110 + max(left, right), +36
  budget bar, +20 waived row, +16 last-entry date, cap 560. Undershoot = scroll flicker, so
  stay generous. Scrollbar chrome is hidden with `scrollbar-width:none`.

Trello still reloads the face iframe on every card-data change (a brief blink). That is
Trello's behavior and cannot be disabled; the mitigations are fewer saves and no pre-paint
overflow. Next lever if asked: only save on explicit Save.

## 4. Data model (Trello pluginData, 4,096-char cap per card)

`t.set('card','shared', {rt, rt_h})` — always the object form, both keys in one write.

`rt` = `serializeState()`:
```
{ v:1, mode:'ucd'|'nonuc'|'nc', qty:{<rateId>:number}, pp:[{id,url,amount,desc?}],   // desc ≤80 chars, omitted when empty
  budget:number|null, bstat:'undefined'|'calculated'|'shared'|'approved'|'billing',
  wt, we, wm: bool (waive Staff / Equipment / Materials), waived:{<rateId>:qty},
  quoted:{}, qbase:{},            // quote-mode "remain" counters
  bimp: bool,                      // budget was imported from description (blocks re-import)
  hby: {'First L.': hours},        // staff-hours distribution by member (Staff Time only)
  pbs: string|undefined,           // budget status before padlock lock (restored on unlock)
  lt: epoch ms }                   // last history entry time → "Last entry <date>" on face
```
`rt_h` = `serializeHist()`: history entries stripped of snapshots, **byte-budgeted to ~2,300
chars / 40 entries** (oldest trimmed). Entry kinds: `{sum:true, name:'+1.5 h Engineer, …',
by, mode}` (one per explicit Save), `{note:true, name, detail}` (quote saved, budget import,
restore), `{reset:true, delta}`, `{cmt:true, name:<comment ≤160>, by, mode}` (Session Comments, v3.6). Per-edit entries were **retired in v2.9**; `logHist()` is an
intentional no-op kept for call-site compatibility.

Member identity: editor calls `t.member('fullName','username')` → `memberTagFrom()` →
"Steven L." Tagged on Save entries and accumulated into `hby`.

## 5. Rates (hard-coded in `RATES`; `rateOf()` honors per-board overrides from the gear menu)

| id | name | Internal | External | kind | group | min |
|---|---|---|---|---|---|---|
| eng | Engineer | 119/h | **160/h** (was 180, fixed v2.9) | time | Staff Time | 0.5 |
| tech | Technician | 30/h | 45/h | time | Staff Time | 0.5 |
| mach | Machining | 25/h | 33/h | time | Equipment Time | 0.5 |
| laser | Laser | 42/h | 64/h | time | Equipment Time | 0.5 |
| gen1 | General 1 (FDM, SLA) | 6/h | 9/h | time | Equipment Time | 0.5 |
| gen2 | General 2 (Polyjet, MJF) | 11/h | 17/h | time | Equipment Time | 0.5 |
| mp2 | Printer 2 (resin) | 0.60/g·ml | 0.90 | material | Materials | 20 |
| mp1 | Printer 1 (PLA etc.) | 0.20/g | 0.30 | material | Materials | 20 |
| moth | Other | $1 | $2 | material | Materials | 5 |
| prepurch | PrePurchasing/Pass-Thru | at cost | at cost | prepurchase | Supplies | 0 |

`TIER = {ucd:'Internal', nonuc:'External', nc:'No Charge'}`. **No Charge** (v3.6): `rateOf()`
returns 0 for every rate incl. supplies multiplier, so quantities are tracked but total is $0; supply
orders still display their entered amounts. Face status label shows "No Charge". Minimums apply in quote mode (lines below
minimum flag `qminHint`).

## 6. Card face (summary view) — what it shows, in order

1. Total (bold) · per-value copy icon (locked only) · **padlock** (monochrome SVG) · far right:
   budget status label + `Budget $X` when set.
2. Budget progress bar (green / amber >90% / red over) with `% used` and `$ remaining` /
   `Over by $X`.
3. Two columns: **Staff Time + Equipment Time** | **Materials + Supplies**. Group header =
   name + `$subtotal` snug beside it. Lines = `[qty box] Name $rate` (qty box blue when
   non-zero). Supplies lists each order as `[n] ID — $amount` (ID falls back to "Order n").
4. Waived row (green, when any), then `Last entry Mon D, YYYY` bottom-right.
5. Empty card → single line **"No recharge details"**, no panel background, 66 px.

Filled cards get a panel background `#2b2c2f` (sampled to match TF-Message's panel exactly;
`--rt-panel`). Text `#cecfd2`, secondary `#96999e`, line `#46474b`, blue `#579dff`. Light
theme has neutral equivalents. **No emoji anywhere; monochrome line icons only.**

**Padlock**: toggles `bstat` to `billing` (remembering the prior status in `pbs`), saves via
relay. When locked, every value gets a small copy icon: total copies `1071.00`, each line
copies its raw qty, each supply order copies `ID — $amount`, and the Supplies header copies the
whole block (one order per line + `Total — $X`). Unlock restores `pbs`.

## 7. Editor (modal) — conventions

- Header row: Internal/External/No Charge segmented toggle (left), gear + **Save** + **Cancel**
  (right); second row centered: **Enter Quote Mode** + **Convert Session Entries to Quote**.
- **Session Comments** textarea under the header (160-char cap, live "n characters remaining").
  Logged as a `cmt` history entry on **Save** only (Cancel / X-close discard it).
- **Convert Session Entries to Quote** (v3.6): "session" = delta vs `_sessBase` (captured at open,
  refreshed after quote save / reset). Enters quote mode with only the delta loaded and the bill
  reverted to open-time; `state.convFrom` holds the full pre-convert bill so quote **Cancel** puts
  the entries back. Save Quote: `qbase` = open-time qty (remain counters count usage above it),
  budget = already-billed net total + quote total. Disabled when there is no delta.
- No title, no "Saved" chip (retired v2.6). Panel is borderless, `max-width:690px`.
- **Save** = `buildSaveEntry()` (one summary history line from a diff against the baseline
  captured at open) + `saveNow()` + `closeModal`. Refused during quote mode ("Finish quote
  first").
- **Cancel** = restore the deep-copied `_openPayload` snapshot (bill + history) and close.
  Deep copy matters: `serializeState()` returns live references.
- Quote mode: button reads **Exit Quote Mode**; a blue **QUOTE MODE** banner shows; auto-save
  is fully suspended so the face never previews a quote-in-progress; Save Quote / Cancel exit
  it. "New charges only" was removed (v3.1). Roll Back / snapshot restore was removed (v3.1).
- Waivers: three checkboxes (Staff Time, Equipment Time, Materials) with per-rate waived
  quantities; history names them specifically ("waiving Equipment Time, waived Machining
  0 → 2 h").
- Quantity values are shown in bordered boxes (matching the face). Budget rounds to the
  penny. Budget input and status dropdown call `saveCard()` (they didn't, pre-v2.1 bug).
- Footer: Generate Summary (copies rich/plain quote), Export CSV, PrePurchasing Summary,
  History (n) with "Staff hours by member" row on top, Reset all (saves immediately).

## 8. Quick Quote (board button, v3.3–3.5)

`board-buttons` capability → button labeled **TF-Recharge** → `?view=editor&scratch=1`.
In scratch mode: `_scratch=true`, `storeGet` returns nothing and `storeSet` is a no-op
(**ephemeral — nothing is ever read or written**), relay disabled, no `hby` attribution,
forced into quote mode with the banner panel hidden, budget/waivers/history/PP summary/
Export CSV/Cancel/Session Comments/Convert hidden, footer becomes two large buttons **Copy Quote** (= Generate Summary)
and **Reset**; header Save reads **Close**. Banner text is just "QUICK QUOTE". Stale `rtq`/
`rtq_h` board-private keys from v3.3–3.4 may exist on the board; they are inert.

## 9. Budget auto-import from the card description

`maybeImportBudget(t, dataStr)` runs in the connector on every card-back-section build.
Intake cards (JotForm → Zapier) contain `**Max Budget:** $ 10000` in **raw markdown**
(`t.card('desc')` returns markdown, not rendered text — this bit us once). Parser:
```
/Max[\s*_]+Budget[\s:*_\\]*\$?[\s*_\\]*\$?[\s*_\\]*([0-9][0-9,]*(?:\.[0-9]{1,2})?)/i
```
Only markdown symbols, colons, `$` and whitespace may sit between label and digits — never
letters — so a blank field followed by `Target Completion Date 08/31/2029` cannot leak the
date. Users sometimes type `$ $900`; both dollars are tolerated. Rejects 0 and >10M.
Guards: never overwrites an existing budget; sets `bimp` so a later manual clear is not
re-imported; **re-reads `rt` at the last instant and merges only budget fields** into the
fresh copy (fixes a race with a concurrent manual save); writes a history note. Power-Ups
can't run at card creation — import happens on first card-back view, which is effectively
the same from a human's perspective.

## 10. Verification pattern (simulation-first — Steven's standing preference)

Every version was verified headless before shipping: Playwright + Chromium, a tiny local
HTTP server, and a **mock `TrelloPowerUp`** injected in place of the CDN script. The mock
models production faithfully: the summary frame's `t.get` hangs forever and `t.render`
never fires; the connector's `t` works; `t.set` supports the object form; `t.member`,
`t.card('desc')`, `t.signUrl`, `t.closeModal`, `getContext().theme` are stubbed; a parent
page hosts connector + face + editor iframes and a shared `__store` so cross-frame
BroadcastChannel relays can be asserted end to end. Tests exercised: URL-data delivery,
height estimates vs. measured content, lock/unlock relay, per-value copiers (by monkey-
patching `copyText`), Save/Cancel semantics, quote-mode save suppression, history summary
diffs (including the reset-baseline bug), importer race, theme param, scratch isolation.

**The harness was never committed** — it was re-created each chat session. Top backlog item:
land it as `test/run.js` (+ `package.json` with playwright) so verification is one command.
Syntax-check both inline `<script>` blocks separately (`node --check`); the head script is a
second block and a naive extractor only checks the first.

## 11. Non-negotiable conventions

- Bump `VERSION` on every change; mention the deploy ritual (and whether a toggle is needed)
  when handing a build back.
- Monochrome, visually clean UI; match Trello's current neutral palette (sampled, not
  guessed). No emoji. Blue `#579dff` is the only accent.
- Never let quote-in-progress data reach the card face.
- Never write `rt_h` from the face; never truncate pluginData silently — the history is
  byte-budgeted by design, the bill is not.
- Quantities left of names on the face; group `$` beside the header text, not far-right.
- Steven prefers decisions-first replies and deliverables ready for immediate use.

## 12. Version history (abridged)

- 1.0 self-refreshing connector; 1.1 URL-delivered face data; 1.2 face quick-add + relay;
  1.3–1.8 face redesigns (mini-editor → read-only itemized two-column), palette matched to
  Trello; 1.9 empty state, countdown save; 2.0 dual-path close-proof save, persisted history;
  2.1 budget-save bug, reset race, penny rounding; 2.2–2.3 budget import (markdown-tolerant);
  2.4 member attribution + hours distribution; 2.5 Save/Cancel buttons, panel bg, title with
  version; 2.6 header restructure, chip retired, exact panel gray; 2.7–2.8 padlock, per-value
  copiers, status label; 2.9 quote leak fix, theme-in-URL, last-entry date, Save-summary
  history, $160 external rate, importer race; 3.0 reset-baseline bugs, supplies block copy;
  3.1 equipment waivers, quote-mode relabel/banner, roll-back + new-charges removed, flicker
  mitigation; 3.2 pre-boot hiding; 3.3–3.5 board-button Quick Quote, ephemeral; 3.6 No Charge tier,
  supply descriptions (editor/face/copy/quote/CSV, +16 px per described order in the height
  estimate), Session Comments, Convert Session Entries to Quote.

## 13. Backlog / open items

1. Commit the Playwright harness (`test/`) — see §10.
2. Face reload blink on each save is Trello's iframe teardown; if still bothersome, move to
   save-on-explicit-Save only (auto-save off), keeping the close-flush as the safety net.
3. Cross-Power-Up quote insertion into TF-Message (needs a shared pluginData key convention;
   Power-Ups can't read each other's data directly).
4. Supplies: ID is free text; consider a URL-aware display on the face.
5. Light-theme palette values were derived, not sampled from a real light-mode screenshot.

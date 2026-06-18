# Golf Pool Live Dashboard — Build Plan (Claude Code handoff)

## 1. Goal

A static, single-page web app that shows a **fantasy golf pool leaderboard** during PGA
tournaments. The organizer drafts 3 teams of 5 players, shares a link, and everyone watching
sees the three teams' combined scores update live off the public scoreboard.

**The critical requirement for this build:** the app must work **week to week for whatever
tournament is currently being played**, with no code changes. It currently hardcodes the 2026
U.S. Open; that hardcoding must be removed and replaced with automatic detection of the active
tournament.

It is hosted on **GitHub Pages** (this repo). It must remain a static site — no backend, no
build step required, no accounts, no API keys.

## 2. Starting point

There is a working single-tournament version (`index.html`) that already implements most of the
UI and logic below. Use it as the base and refactor it; do not start from scratch unless that is
cleaner. The parts worth preserving are called out in section 6. If `index.html` is not in the
repo, this spec is self-contained enough to build fresh.

## 3. Data source — ESPN public golf API (no key, CORS-open)

ESPN exposes an unofficial-but-stable JSON feed that browsers can read directly (it returns
`Access-Control-Allow-Origin: *`, so it works from GitHub Pages). **Verify the exact response
shapes live before coding** — curl these and inspect:

```
# Season calendar + (sometimes stale) "current" scoreboard
https://site.api.espn.com/apis/site/v2/sports/golf/pga/scoreboard

# Full-field leaderboard for ONE specific event
https://site.api.espn.com/apis/site/v2/sports/golf/pga/leaderboard?event={EVENT_ID}
```

### Known quirk (this is why we use the calendar)
The default `/scoreboard` endpoint does **not** reliably point at the current week's event — it was
observed serving the *previous* week's tournament during a live major. **Do not trust it to tell
you which tournament is "current."** Instead, derive the current event from the calendar (below).

### Response contract (verified fields)
From either endpoint's top-level JSON:

- `leagues[0].calendar[]` — the full season schedule. Each entry:
  - `id` — the event ID (e.g. `"401811952"`)
  - `label` — tournament name (e.g. `"U.S. Open"`)
  - `startDate`, `endDate` — ISO strings in UTC (e.g. `"2026-06-18T07:00Z"` / `"2026-06-21T07:00Z"`)

From the leaderboard-for-an-event response:

- Competitors live at `events[0].competitions[0].competitors[]` (defensive: also check
  `events[].competitors[]` and `competitions[0].competitors[]` — extractor should try all three).
- Per competitor:
  - `athlete.displayName` — e.g. `"J.T. Poston"` (also `athlete.fullName` as fallback)
  - `score` — tournament total **to par**, as a string: `"-12"`, `"E"`, `"+1"`, or empty
  - `linescores[]` — one entry per round; each has `displayValue` (that round's to-par, e.g.
    `"-2"`), `value` (strokes), and `period` (1–4)
  - `status` — live state; may include `thru` (holes done), `period` (current round), and
    `type.shortDetail` / `type.detail` (e.g. `"In Progress"`, `"CUT"`, `"F"`). Fields vary, so
    read defensively.
- Tournament name/venue for the header: `events[0].name` and, if present,
  `events[0].competitions[0].venue.fullName` (fall back to the calendar `label`).

## 4. Core feature: automatic tournament detection

On load and on each refresh:

1. Fetch the scoreboard endpoint and read `leagues[0].calendar[]`.
2. Pick the active event:
   - **In progress:** the event where `startDate − 12h ≤ now ≤ endDate + 18h`
     (the grace window keeps Thursday-morning and Sunday-evening views on the right event).
   - **Else upcoming:** the nearest future event (smallest `startDate > now`). Show a
     "Tees off {date}" note; scores will be empty/dashes until play starts.
   - **Else** (off-season edge): the most recently finished event.
3. Use that event's `id` to fetch `/leaderboard?event={id}` and render.
4. Cache the detected event id in memory for the session; re-detect on a manual "change
   tournament" action or naturally on next load.

Display the detected tournament name (and course if available) in the header instead of the
hardcoded "U.S. Open · Shinnecock Hills". Show the current round number when known
(`status.period` or the max `linescores.period` across the field).

> The roster (teams/players) is **independent of the tournament.** Each week the organizer edits
> the 5-man teams and shares a fresh link; the app should never tie a stored roster to a specific
> event id.

## 5. Functional requirements

### Scoring
- All scores are **to par**: negative = under (good), `0` = even (display "E"), positive = over.
- Parse the `score` / `displayValue` strings to integers: `"E" → 0`, strip leading `+`, handle
  `"-"`/empty → `null` (unknown).
- **Team Total** = sum of its 5 players' tournament to-par totals.
- **Team Today** = sum of its 5 players' current-round to-par (the in-progress / most recent
  round). This is the "combined score for the day."
- **Standings:** rank the 3 teams by Total ascending (lowest wins). Teams with no data yet sink
  to the bottom. Show a movement indicator (▲/▼) comparing to the previous refresh.

### Player name matching (must be forgiving)
Users type first + last names; ESPN uses `displayName`. Normalize both sides before matching:
lowercase, strip diacritics (NFD + remove combining marks), remove `.`, `'`, `’`, backticks,
collapse whitespace. Match exact-normalized first; if no hit, fall back to a substring match
(either contains the other) to catch "JT Poston" ↔ "J.T. Poston". Any player that still can't be
matched must be visibly flagged in the UI (e.g. red row, "name not matched") so the organizer can
fix the spelling.

> Note for non-major weeks: LIV Golf players are usually absent from regular PGA Tour fields and
> will legitimately show "name not matched." This is expected, not a bug.

### Edge cases
- **Between tournaments / before tee-off:** show the upcoming event name + tee-off date, teams
  visible with dashes, no error.
- **Made/missed cut (after R2):** ESPN freezes a cut player's total; display it as-is, and show
  their status (e.g. "CUT") in the player row. Their frozen total still counts toward the team.
- **Player not in field:** treat as `null` (don't crash team sums; `null` contributes nothing).
- **Feed unreachable:** show a clear, non-technical error and keep the last good scores on screen
  if any.

### Persistence & sharing (preserve current behavior)
- **Persistence:** save the roster to `localStorage` so the organizer's device remembers teams
  between visits. (This is a real-browser static site, so `localStorage` is correct here.)
- **Share link:** a "Share" button encodes the current roster (JSON → base64 → `encodeURIComponent`,
  unicode-safe) into the URL hash as `#r=...`, builds `location.origin + location.pathname + "#r=" + enc`,
  and copies it to the clipboard (with a textarea fallback for browsers that block the Clipboard API).
- **Load priority:** on boot, if the URL hash has `#r=`, decode it, use it, and also persist it to
  localStorage; otherwise load from localStorage; otherwise show the setup screen.
- The shared link is the source of truth for teams. A viewer editing teams locally only changes
  their own copy.

### Refresh
- Manual "Refresh scores" button.
- "Auto-refresh" toggle, default off; when on, poll every ~2 minutes. Re-run tournament detection
  + leaderboard fetch each cycle. Be respectful of the feed (no tighter than ~60s).

## 6. Parts of the existing app to keep

- The to-par formatting + color coding (under = green, over = red, E = neutral).
- The collapsible team rows (tap a team to see its 5 players, holes-thru, individual scores).
- The standings reorder + ▲/▼ movement arrows.
- The setup screen with the 3 teams × 5 players inputs and editable team names; yellow-highlighted
  input cells.
- The multi-URL fetch fallback and the defensive competitor extractor.
- The visual identity (fairway-green background, parchment cards, flag-red accent) — keep it; just
  make the title text dynamic.

## 7. Optional config (build if low-cost; otherwise leave clean hooks)

Pools vary week to week. Expose these as simple constants or a small settings UI:

- **Scoring mode:** `total of all 5` (default) vs **best 4 of 5** (drop each team's worst score).
- **Cut penalty:** none (default) vs a fixed stroke penalty added to cut players.
- **Team count / team size:** currently fixed at 3 × 5; keep these as constants so they're easy to
  change, even if the UI stays 3 × 5 for now.
- **Refresh interval.**

Don't over-engineer — clean constants are fine if a full settings panel is out of scope.

## 8. Suggested structure

Single-file `index.html` (HTML + CSS + JS inline) is acceptable and keeps GitHub Pages trivial.
If you prefer separation, `index.html` + `styles.css` + `app.js` is fine — but **the entry file
must be `index.html` at the repo root** so GitHub Pages serves it at the clean URL.

Recommended JS responsibilities (names illustrative):
- `detectCurrentEvent(calendar, now)` → `{ id, label, state }`
- `fetchLeaderboard(eventId)` → competitors
- `parToInt(str)`, `normName(str)`
- `buildScoreMap(competitors)` → `{ normalizedName: {total, today, thru, status} }`
- `matchPlayers(roster, scoreMap)` → per-player scores + unmatched flags
- `aggregate(teams, scores, mode)` → team totals + standings
- roster `load/save`, `encodeRoster/decodeRoster`, `shareLink`
- `render()` for setup vs board

## 9. Deployment (GitHub Pages — already in use)

- `index.html` at repo root.
- Settings → Pages → Deploy from branch → `main` / root.
- Verify the live `https://<user>.github.io/<repo>/` URL loads, detects the current event, and the
  Share button produces a working `#r=` link.

## 10. Non-goals

- No backend, server, database, or websockets.
- No authentication or per-user accounts.
- No paid/keyed data providers (ESPN public feed only).
- No tick-by-tick streaming — periodic polling is sufficient.
- Not affiliated with the USGA/PGA TOUR/ESPN; it's a private pool tool. Keep a small disclaimer in
  the footer.

## 11. Acceptance criteria (test these before calling it done)

1. **Tournament-agnostic:** with the current week's tournament live, the app detects it from the
   calendar (not hardcoded) and shows the correct event name and field.
2. **Stale-scoreboard resilience:** confirm detection still picks the right event even if the
   default `/scoreboard` returns a different (older) tournament.
3. **Scores correct:** spot-check 3 real players against the public leaderboard — total, today,
   and thru match.
4. **Name matching:** accented names (e.g. "Aberg" → "Åberg") and punctuated names ("JT Poston" →
   "J.T. Poston") match; a deliberately misspelled name flags as unmatched.
5. **Standings:** lowest combined total ranks first; movement arrows change after a refresh that
   alters order.
6. **Share round-trip:** Share copies a link; opening that link in a fresh browser/incognito loads
   the exact teams and live scores with no setup.
7. **Persistence:** reload the page — teams are remembered.
8. **Between-tournament state:** simulate a date with no live event; app shows the upcoming event
   and tee-off date without errors.
9. **Offline/feed-down:** block the network; app shows a clear message and doesn't crash.
10. **Static-host clean:** works from the GitHub Pages URL with no console errors.

## 12. Notes for week-to-week operation (organizer workflow)

Each tournament week the organizer: opens the hosted URL → **Edit teams** → enters that week's 15
players → **Save** → **Share** → posts the new link. No deploys, no code edits. Consider adding
these operational facts to the repo's `CLAUDE.md` so future Claude Code sessions retain the
multi-tournament intent and the ESPN data contract.

# CLAUDE.md

## What this is
A static, single-page **golf pool live dashboard**, hosted on GitHub Pages. An organizer drafts
3 teams of 5 PGA players, shares a link, and everyone watching sees the teams' combined to-par
scores update live. Full spec lives in `PLAN.md` — read it before making changes.

## Non-negotiable invariants
- **Static site only.** No backend, no build step, no accounts, no API keys. Must run from the
  GitHub Pages URL as-is. Entry file is `index.html` at repo root.
- **Tournament-agnostic.** Never hardcode a tournament/event. The app must auto-detect the
  current PGA event every week (see Data section). This is the core reason the app exists weekly.
- **Roster is independent of the tournament.** Each week the organizer re-enters teams and shares
  a new link. Do not bind a stored roster to an event id.
- **Roster travels in the URL.** Share encodes the roster into the `#r=` hash (unicode-safe
  base64). The shared link is the source of truth for teams. `localStorage` persists the
  organizer's own copy between visits (correct here — this is a real-browser page, not a Claude
  artifact).

## Data source — ESPN public golf API (no key, CORS-open)
Endpoints:
```
https://site.api.espn.com/apis/site/v2/sports/golf/pga/scoreboard          # has season calendar
https://site.api.espn.com/apis/site/v2/sports/golf/pga/leaderboard?event={ID}  # one event's field
```
**Gotcha (do not forget):** the default `/scoreboard` is unreliable for "which tournament is
current" — it has served the *previous* week's event during a live major. **Always detect the
current event from `leagues[0].calendar[]`** (each entry: `id`, `label`, `startDate`, `endDate`
in UTC), then fetch that event's leaderboard by id.

Detection rule: in-progress event = `startDate − 12h ≤ now ≤ endDate + 18h`; else nearest
upcoming; else most recent past.

Leaderboard field contract (read defensively, shapes vary):
- competitors at `events[0].competitions[0].competitors[]` (also try `events[].competitors`,
  `competitions[0].competitors`).
- `athlete.displayName` (fallback `fullName`); `score` = tournament to-par string
  (`"-12"`/`"E"`/`"+1"`); `linescores[]` per round with `displayValue` (round to-par) + `period`;
  `status` with `thru`, `period`, `type.shortDetail`/`detail` (e.g. `"CUT"`, `"F"`).

## Scoring rules
- Everything is **to par**: `"E" → 0`, strip `+`, `"-"`/empty → `null`.
- Team **Total** = sum of 5 players' tournament to-par. Team **Today** = sum of 5 players'
  current-round to-par. Standings rank by Total ascending (lowest wins).
- Default scoring = total of all 5. Keep clean hooks for optional **best 4 of 5** and a cut
  penalty (see `PLAN.md` §7). Team count/size (3 × 5) are constants, not magic numbers.

## Player name matching (keep forgiving)
Normalize both sides: lowercase, strip diacritics, remove `.` `'` `’` backtick, collapse spaces.
Exact-normalized match first, then substring fallback ("JT Poston" ↔ "J.T. Poston"). Flag any
unmatched player visibly so the organizer can fix spelling. LIV players absent from regular
fields legitimately won't match — expected, not a bug.

## Known edge cases to keep handling
Between tournaments / pre-tee-off (show upcoming event + date, no error); missed cut (frozen
total still counts, show "CUT"); player not in field (`null`, don't break sums); feed unreachable
(clear message, keep last good scores).

## Operator workflow (per week)
Open hosted URL → Edit teams → enter that week's 15 players → Save → Share → post the new link.
No deploys, no code edits.

## Before shipping changes
Run the acceptance checks in `PLAN.md` §11. The easiest one to overlook and the most important:
confirm tournament detection lands on the correct event **even when `/scoreboard` is stale** —
that bug would silently show last week's tournament.

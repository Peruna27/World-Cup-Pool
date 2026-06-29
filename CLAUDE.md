# World Cup Pool — 2026 FIFA World Cup Tracker

A mobile-first fantasy tracker for the 2026 FIFA World Cup auction pool with 4 friends (Mike, Joe, Reed, Will).

> **READ THIS BEFORE EDITING.** The pool rules and data flow have non-obvious quirks documented here.

---

## Pool Rules

- **4 owners**: Mike, Joe, Reed, Will. Each had a $100 auction budget.
- **Winner = whoever owns the team that wins the World Cup.** Winner-take-all.
- **17 named teams** were auctioned. The remaining 31 teams ("the Field") were bought by Mike for $29. So Mike owns 7 named teams + the Field.
- **Golden Boot side pool**: each owner randomly drew 5 players. Whoever owns the tournament's top scorer (FIFA Golden Boot) wins the side pot.

### Team ownership (with auction cost)

| Owner | Spent | Teams |
|---|---|---|
| Joe | $100 | France $78, Colombia $10, Mexico $5, Japan $5, Sweden $2 |
| Mike | $100 | Brazil $13, Norway $11, Argentina $13, Netherlands $13, Germany $12, Belgium $7, Morocco $2, **Field $29** |
| Reed | $100 | England $60, Portugal $40 |
| Will | $100 | Spain $81, USA $13, Switzerland $6 |

### Golden Boot rosters

| Joe | Mike | Reed | Will |
|---|---|---|---|
| Igor Thiago | Haaland | Wirtz | Dembélé |
| C. Ronaldo | Woltemade | Mbappé | F. Torres |
| Vini Jr | Havertz | Oyarzabal | Gakpo |
| Kane | Yamal | Raphinha | Lukaku |
| Olise | Lautaro Martínez | J. Álvarez | Messi |

---

## Tournament Facts

- **2026 FIFA World Cup**: June 11 – July 19, 2026
- **Hosts**: USA, Canada, Mexico
- **48 teams**, 12 groups of 4 → Round of 32 (top 2 from each group + best 8 third-placed teams) → R16 → QF → SF → Final
- **104 total matches** (100 fixtures + final)
- **ESPN event slug**: `soccer/fifa.world`

---

## Data Sources

### ESPN (fixtures, scores, group standings, per-match odds)

```
GET https://site.api.espn.com/apis/site/v2/sports/soccer/fifa.world/scoreboard?dates=20260611-20260719&limit=300
```

**`&limit=300` is required** — the default caps at 100 events and silently drops the last 4 (both semis, 3rd-place, final). With it you get all 104.

Returns 104 events. Each event has:
- `competitions[0].competitors[]` with `team.displayName`, `score`, `homeAway`
- `season.slug` = the round: `group-stage`, `round-of-32`, `round-of-16`, `quarterfinals`, `semifinals`, `3rd-place-match`, `final`. **The bracket reads straight from this** — ESPN fills in resolved knockout teams itself (all seeding + best-third-place math), so we do NO bracket calculation. Undecided slots arrive as placeholder names (`"Group L Winner"`, `"Third Place Group E/H/I/J/K"`, `"Round of 32 1 Winner"`, `"Semifinal 1 Loser"`).
- `competitions[0].status.type` with `state` (`pre`/`in`/`post`), `shortDetail` (e.g. "FT", "85'", "Half")
- `competitions[0].odds[]` with DraftKings moneyline + over/under per game
- `competitions[0].venue.fullName`
- `competitions[0].details[]` = scoring/card plays. Goals have `scoringPlay: true` and the scorer in `athletesInvolved[0]` (only the scorer — no assists). **Golden Boot tally must exclude `ownGoal` and `shootout`** (own goals aren't credited; shootout pens don't count); in-match `penaltyKick` goals DO count. This feeds `goalScorers` (roster tally) + `goalLeaders` (all scorers) — `goalScorers` is keyed by roster player name via `matchPlayer()`, so it stays empty unless `parseGoalScorers()` actually runs in `fetchAll`.

Group standings:
```
GET https://site.api.espn.com/apis/v2/sports/soccer/fifa.world/standings
```

### Polymarket (championship probabilities)

The "World Cup Winner" event at `gamma-api.polymarket.com/events?slug=world-cup-winner` has ~60 individual markets like *"Will Spain win the 2026 FIFA World Cup?"*. Each has `outcomePrices` (a JSON string) with "Yes" probability in `[0]`.

Parse the team name out of the question with `/^Will\s+(.+?)\s+win/i` and aggregate per owner.

**Golden Boot odds ARE now live on Polymarket** at `gamma-api.polymarket.com/events?slug=world-cup-golden-boot-winner` (~80 markets like *"Will Lionel Messi be the top scorer..."*). Parsed by `parsePolymarketBoot` with `/^Will\s+(.+?)\s+be\s+the\s+top/i` into `playerOdds` + `playerOddsList`. Boot cards show **both** live goal counts (from ESPN `details`) and these odds; Top Scorers shows live goals once any are scored, else falls back to these odds pre-tournament.

### Owner odds calculation

```
Joe's odds   = sum of (Polymarket P) for Joe's teams
Reed's odds  = same
Will's odds  = same
Mike's odds  = sum of Mike's named teams + Field
             = sum of Mike's named teams + (1 - sum of ALL named teams across all owners)
```

The 100% across all 4 owners must sum to 1 (any team must win). If Polymarket prices don't sum to exactly 1 (they often don't due to vig and rounding), the Field bucket absorbs the slack — which lands on Mike. That's fine because that's exactly what Mike paid for.

---

## Team Name Aliases

Polymarket and ESPN sometimes disagree on country names. Each team must resolve to **one canonical name** (the spelling in `GROUPS` / owner rosters) via `TEAM_ALIASES`. `displayTeam()` then renders that one name everywhere so the same team never appears under two spellings.

**Confirmed from the live ESPN feed (2026-06-16), these are the spellings ESPN actually sends that differ from our canonical:**
- `Türkiye` → **Turkey**  ← the one that broke group standings; FIFA renamed Turkey in 2022
- `Bosnia-Herzegovina` → **Bosnia and Herzegovina**
- `United States` → **USA**
- `Czechia` → **Czech Republic**
- `Congo DR` → **DR Congo**

Other aliases (`Korea Republic`, `IR Iran`, `Cabo Verde`, `Côte d'Ivoire`) are safety nets — ESPN currently sends the canonical form directly, but Polymarket/FIFA may not.

Always normalize names through `name.toLowerCase().normalize('NFD').replace(/[̀-ͯ]/g, '')` to handle accents (e.g. `Ousmane Dembélé` ↔ `dembele`). **`TEAM_ALIASES` keys must be lowercase + accent-stripped** — `normalizeTeam()` strips accents *before* the lookup, so an accented key (e.g. `côte d'ivoire`) can never match and is dead code.

---

## Layout

Single-page app, mobile-first:

1. **Header** — sticky pitch-green band with title + refresh.
2. **Pool Standings** — 4 owner cards sorted by current win probability. Tap to expand and see each owner's teams with their individual probabilities + cost paid.
3. **Golden Boot** — 4 owner cards sorted by their leading player's goal count. Tap to expand and see all 5 players + goal counts.
4. **Tabs**: Fixtures · Groups · Bracket
   - **Fixtures**: all 104 games grouped by date, click any row to open match details. Pool teams get color tags.
   - **Groups**: 12 group tables, top 2 highlighted (advance directly to R32).
   - **Bracket**: connected R32 → Final tree with elbow connector lines. Teams + results come 100% from ESPN (it does all the seeding / best-third-place math). BUT ESPN's scoreboard does NOT give the bracket wiring, and its `"Round of 32 N Winner"` labels number games by **official match order (M73 = R32 game 1), NOT by calendar date** — mapping by date silently scrambles the bracket (was a real bug). So each fixture is pinned to its official position: R32 via `BRACKET_R32_SLOTS` (the published group-slot layout, `'1D'` = winner Group D, matched on each fixture's winner/runner-up side), and R16+ via `KO_FEED` (which games feed which). **`KO_FEED` is verified against the published 2026 bracket** ([Wikipedia](https://en.wikipedia.org/wiki/2026_FIFA_World_Cup_knockout_stage)) — note the QF crossover (`QF2 ← R16 5&6`, `QF3 ← R16 3&4`) that FIFA uses so e.g. winners of Groups D & I meet in the **semifinals**, not the final. `KO_ORDER` (in-order tree walk of `KO_FEED`) re-stacks each round so a game's two feeders sit directly above/below it, which is what makes the pure-CSS connectors (`.bracket-slot::before/::after`, sized to the column gap) line up. 3rd-place match renders separately, outside the tree.
5. **Match modal** — tap a fixture row for score, time, venue, owner tags, and per-game odds.

---

## Owner Colors (consistent across the app)

- Joe → red `#e74c3c`
- Mike → blue `#3498db`
- Reed → gold `#d4a017`
- Will → green `#27ae60`

---

## Hosting

- GitHub repo: `Peruna27/World-Cup-Pool` (user creates on GitHub)
- GitHub Pages live URL: `https://peruna27.github.io/World-Cup-Pool/`
- Static HTML/JS, no build, no backend. Push to `main` → Pages rebuilds in ~60s.
- Git identity: `Local User <local@local>` (placeholder).

---

## Known Pitfalls (will hit these next time)

- **Polymarket markets sum to >1**: vig is built in (e.g. all "Yes" prices total ~1.10). If you want true probabilities, normalize by dividing each by the sum. For Mike's Field calculation, this means the Field probability uses the un-normalized remainder, which is roughly fair.
- **CORS**: ESPN and Polymarket public APIs allow browser CORS. If either changes, we'd need a CORS proxy or a backend.
- **Inactive markets**: Polymarket has some `active: false` markets for teams that didn't qualify. Skip those in the parser (already done).
- **Browser concurrent connection limit**: Don't fire 100+ per-event status requests in parallel. Batch in 6–8 (same lesson from Golf-League's tee-time fetch).
- **Bracket structure**: 2026 World Cup uses a Round of 32 (new for 48-team format). Don't assume Round of 16 is the start of knockouts.
- **Names with accents**: always `normalize('NFD').replace(/[̀-ͯ]/g, '')` before matching.
- **One bad team name silently breaks a whole group**: `deriveGroups()` only tallies a finished fixture when BOTH names match a group row (`if (!home || !away) continue`). An unmatched name drops the entire game, so the team *and its opponents* lose those points. Sanity check: `sum of all teams' GP` must equal `2 × (completed group games)`. If it doesn't, a name isn't matching — add it to `TEAM_ALIASES`. (This is exactly how `Türkiye` broke Group D.)
- **Knockout games must NOT count toward group standings**: every knockout team still belongs to a group, so a result like "South Africa 0-1 Canada" in the R32 matches BOTH group rows and (without a guard) gets tallied into the group table — handing Canada phantom points and flipping the group order, which then mis-seeds the bracket. `deriveGroups()` MUST filter `if (f.round !== 'group-stage') continue`. Don't rely on `if (!home || !away)` to exclude knockouts — it doesn't.
- **Bracket advancement**: once an R32 game finishes, ESPN swaps the next round's placeholder (`"Round of 32 5 Winner"`) for the real team, so the feeder-ref parse returns null for that side. `bracketFixturesByPosition` therefore also maps an advanced team back to its R32 position (`r32PosByTeam`) so the R16+ box can still be placed. Winner/loser styling uses ESPN's per-competitor `winner` flag (shootout-safe), not the score.
- **Mike owns the Field**: his card must show the Field as a "team" with its computed probability. Don't omit it just because it's synthetic.

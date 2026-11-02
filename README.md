# N8n Betting Workflow

Automated football odds scanner that uses an LLM to post value-bet tips to Telegram and Discord.

An n8n workflow (`sports betting agent.json`) that pulls football (soccer) odds and fixtures every 6 hours, asks an LLM to analyse each match, filters for "value bets", and posts the tips to Telegram and Discord.

> **Disclaimer:** the analysis is based on bookmaker odds only. It does not use form, injuries or any other stats, and it is not financial or betting advice.

## How it works

```
Manual Trigger ─┐
                ├─► Get Betting Odds ──► Map Odds ──┐
Every 6 hours ──┤                                   ├─► Merge Data Sources ─► Process All Data ─┬─► Expert Betting Agent  ─► Format Betting Tips  ─► Filter Value Bets  ─► Discord
                └─► Get Match Statistics ───────────┘                                           └─► Expert Betting Agent1 ─► Format Betting Tips1 ─► Filter Value Bets1 ─► Telegram
```

| Step | Node | What it does |
|------|------|--------------|
| 1 | **Manual Trigger / Schedule (6h)** | Starts a run. Both feed the two data fetches in parallel. |
| 2 | **Get Betting Odds** | Calls [The Odds API](https://the-odds-api.com) `soccer_epl/odds` (regions `uk,eu`, market `h2h`). |
| 3 | **Get Match Statistics** | Calls [football-data.org](https://www.football-data.org) `/v4/matches` for today to +3 days (competitions `PL`, `CL`, `FL1`). Used only for kickoff times. |
| 4 | **Map Odds** | Per match, takes the first bookmaker price for home/draw/away and counts bookmakers. |
| 5 | **Merge Data Sources** | Combines the odds items and the fixtures response into one stream. |
| 6 | **Process All Data** | Normalises team names (accents, FC/AFC suffixes), matches odds to football-data fixtures to get an accurate kickoff time, and takes the **best (highest) price across all bookmakers** for each outcome. Outputs `matchId`, `homeTeam`, `awayTeam`, `league`, `matchTime`, `homeOdds`, `drawOdds`, `awayOdds`, `bookmakerCount`. |
| 7 | **Expert Betting Agent / Agent1** | LangChain agents (OpenRouter, `meta-llama/llama-3.3-70b-instruct:free`) with a window memory of 10 keyed by home team. They compute the math below and return strict JSON. |
| 8 | **Format Betting Tips / Tips1** | Extracts JSON from the agent reply (also handles code fences), merges it with the original match data, computes `isValueBet`, and builds display strings (UK-time kickoff, odds line, EV %, risk label). On any error it emits a safe "Processing Error" item instead of failing. |
| 9 | **Filter Value Bets / Bets1** | IF node that lets a tip through to the alert. |
| 10 | **Send a message (Discord)** / **Send Telegram Alert** | Posts the formatted tip. |

There are two parallel branches: the first ends in **Discord**, the second (`...1` nodes) in **Telegram**. They use different prompts (see below).

### The analysis the agents perform

All numbers come from the input odds:

1. Implied probability `1/odds` for each outcome, normalised by the overround `Z` to remove the bookmaker margin.
2. `prediction` = outcome with the highest probability; `confidence` = `round(100 × max p)`.
3. `expectedValue` (%) = `100 × (p_pick × odds − 1)`.
4. `stake` in 0–3 units from the EV band.
5. `riskLevel` 1–10 starting at 6, adjusted by market depth (bookmaker count), EV, and so on.

The Discord agent's prompt is the richer one. It adds Kelly fraction, margin/overround, a market-efficiency score, a value rating and a mathematical summary. The Telegram agent's prompt is a shorter version with the same core formulas.

### Value-bet rule

`isValueBet = confidence >= 65 && expectedValue > 5`

## Setup

1. Import `sports betting agent.json` into n8n (**Workflows → Import from file**).
2. Create credentials and attach them to the nodes:
   - **OpenRouter** (`openRouterApi`) on both *OpenRouter Chat Model* nodes.
   - **Telegram** (`telegramApi`) on *Send Telegram Alert*, and set your own `chatId`.
   - **Discord OAuth2** (`discordOAuth2Api`) on *Send a message*, and set your own guild and channel IDs.
3. Get your own keys and enter them in the HTTP nodes:
   - The Odds API key in *Get Betting Odds* (`apiKey` query parameter).
   - football-data.org: the node currently sends **no** `X-Auth-Token` header. Add one if the request is rejected.
4. Run **Start Here - Manual Test**, check each node's output, then activate the workflow so the 6-hour schedule runs.

To cover other leagues, change `soccer_epl` in the odds URL and the `competitions` values in the stats request.

## Security: fix before sharing or publishing

The exported JSON contains values that should not be public:

- A **live Odds API key** is hardcoded in *Get Betting Odds*. Revoke and regenerate it, and move it to an n8n credential or environment variable.
- A Telegram `chatId` and Discord guild/channel IDs are hardcoded.

## Known issues found while reading the workflow

- **`confidenceBar` is never produced.** Both message templates reference `$json.display.confidenceBar`, but *Format Betting Tips* doesn't build it, so that part of the message is blank.
- **The filter does not match the documented rule.** The IF nodes use `combineOperation: any` (OR), so a tip passes if confidence ≥ 65, or EV > 5, or `isValueBet` is true. The rule above needs AND.
- **The EV > 5% threshold is rarely reachable.** EV is computed from normalised probabilities taken from the bookmakers' own odds, so it is usually within a few percent of zero. The agents' own prompts treat EV above 3% as suspiciously high. Expect few or no alerts at the current thresholds.
- **Sticky note is out of date.** It mentions OpenAI, Google Sheets logging and a "Risk Level ≤ 7" condition. None of these exist in the workflow (it uses OpenRouter, Discord and Telegram, and has no risk filter).
- **Two nodes disagree on prices.** *Map Odds* takes the first bookmaker's price, while *Process All Data* takes the best price across bookmakers. Only *Process All Data* feeds the agents.
- **Football-data request:** it passes `competitions` three times, and it is only used for kickoff times. Leagues other than EPL have no odds data, so they never produce tips.
- **Agent memory is keyed by home team**, so earlier matches for the same team are fed back into the context of later runs.
- **Alert text is in transliterated Moroccan Darija** (e.g. "BETTING TIP 3LA 9LEB"). Edit the message templates for another audience.
- **Free model.** `llama-3.3-70b-instruct:free` is rate limited and may not always return valid JSON. The formatter degrades gracefully, but expect some "no prediction" results.
- **Duplication.** The two branches are near copies of each other. They could be collapsed into one agent feeding both messengers.
# N8n-Betting-Workflow

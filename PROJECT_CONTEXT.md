# Booker — Multi-City Flight Search Agent

## What is Booker?

Booker is a CLI tool that finds cheap flights with a **stopover** — a 2-6 day city visit between origin and destination. Instead of flying DEL → YYZ directly, it searches routes like DEL → Istanbul (explore for 4 days) → YYZ, often finding significantly cheaper fares.

It uses Google Flights data (via SerpAPI), an LLM for intelligent ranking, and disk caching to avoid burning API calls during development.

## How It Works

The search pipeline runs in 5 stages:

```
EXPAND → FETCH → FILTER → COMBINE → RANK
```

1. **EXPAND**: For a given origin/destination, generate search requests for each candidate stopover city (e.g. Hong Kong, Istanbul, Singapore — 10 cities for DEL→YYZ).

2. **FETCH**: Two strategies run in parallel per city:
   - **One-way legs**: Search origin→stopover and stopover→destination separately, combine later.
   - **Native multi-city**: Use Google Flights' multi-city mode (SerpAPI type=3), which returns combined pricing — often much cheaper than 2× one-way.

3. **FILTER**: Remove blocked airlines/hubs (e.g. Middle East airspace closures), zero-price artifacts, flights exceeding max layovers, and flights outside the date window.

4. **COMBINE**: Pair leg1 + leg2 where the gap between arrival and departure is a valid stopover (2-6 days). Merge one-way combinations with native multi-city results.

5. **RANK**: Send top 15 candidates (by price) to an LLM, which scores each 0-100 based on configurable weighted criteria: cost, airline consistency, layover quality, flight duration, stopover city appeal, and schedule convenience.

## Quick Start

### Prerequisites
- Go 1.23+
- API keys for SerpAPI, and either Anuma or OpenAI (for LLM ranking)

### Setup

1. Clone and install:
   ```bash
   git clone https://github.com/kingpinXD/booker.git
   cd booker
   make install
   ```

2. Create a `.env` file with your API keys:
   ```
   BOOKER_SERPAPI_KEY=your_serpapi_key
   BOOKER_ANUMA_API_KEY=your_anuma_key      # Primary LLM (optional)
   BOOKER_OPENAI_API_KEY=your_openai_key    # Fallback LLM
   BOOKER_KIWI_API_KEY=your_kiwi_key        # Optional secondary provider
   ```

3. Run a search:
   ```bash
   booker search DEL YYZ --date 2026-03-24 --leg2-date 2026-03-30
   ```

### CLI Flags

```
booker search <origin> <destination> --date YYYY-MM-DD --leg2-date YYYY-MM-DD [flags]

Required:
  --date         Leg 1 departure date
  --leg2-date    Leg 2 departure date (when to leave the stopover city)

Optional:
  --passengers   Number of passengers (default: 1)
  --cabin        Cabin class: economy, premium_economy, business, first (default: economy)
  --flex-days    Date flexibility in days (default: 3, searches ±3 days around target)
  --max-stops    Max layovers per leg: -1 = no limit, 0 = direct only (default: -1)
  --max-results  Number of top results to display (default: 5)
  --profile      Ranking profile: budget, comfort, balanced (default: budget)
  --currency     Display currency: CAD, USD, EUR, GBP, INR, etc. (default: CAD)
```

### Example Output

```
╭───┬───────┬────────┬─────────────────────────┬────────────────┬───────────────────────┬─────────────────┬─────────────────┬───────────────────╮
│ # │ SCORE │ PRICE  │ ROUTE                   │ LEG 1 AIRLINES │ LEG 2 AIRLINES        │ LEG 1 DEPARTURE │ LEG 2 DEPARTURE │ STOPOVER          │
├───┼───────┼────────┼─────────────────────────┼────────────────┼───────────────────────┼─────────────────┼─────────────────┼───────────────────┤
│ 1 │ 78    │ C$1511 │ DEL→BOM→RKT→IST→FRA→YYZ │ IndiGo         │ Lufthansa             │ Mar 24 23:30    │ Mar 30 05:50    │ Istanbul (4d 17h) │
│ 2 │ 75    │ C$1568 │ DEL→BOM→RKT→IST→FRA→YYZ │ IndiGo         │ Lufthansa             │ Mar 24 00:30    │ Mar 30 05:50    │ Istanbul (5d 17h) │
│ 3 │ 72    │ C$1589 │ DEL→BOM→RKT→IST→FRA→CDG→YYZ │ IndiGo     │ Lufthansa, Air Canada │ Mar 24 23:30    │ Mar 30 05:50    │ Istanbul (4d 17h) │
╰───┴───────┴────────┴─────────────────────────┴────────────────┴───────────────────────┴─────────────────┴─────────────────┴───────────────────╯
```

The route column shows the full path including connections within each leg, so you can see exactly where you'll stop.

## Project Structure

```
booker/
├── main.go              # Entry point
├── Makefile             # install, build, test, clean
├── cmd/                 # CLI (cobra + viper)
│   ├── root.go          # Root command, .env loading
│   └── search.go        # Search command, flags, table output
├── config/              # Configuration, API constants, env vars
├── types/               # Normalized types: Flight, Segment, Money
├── provider/            # Flight data providers
│   ├── provider.go      # Provider + MultiCitySearcher interfaces, Registry
│   ├── serpapi/         # Google Flights via SerpAPI (primary)
│   ├── kiwi/            # Kiwi via RapidAPI (secondary)
│   └── cache/           # Disk-based caching decorator
├── httpclient/          # Shared HTTP client with retries
├── llm/                 # LLM client (Anuma primary, OpenAI fallback)
├── currency/            # Live exchange rates
├── search/              # Core search types + filters
│   └── multicity/       # Search orchestrator, combiner, LLM ranker
├── aggregator/          # Generic multi-provider fan-out (available, unused)
└── .cache/flights/      # Cached API responses (gitignored)
```

## Key Design Decisions

- **Dual search strategy**: One-way combination finds more options; native multi-city gets better prices. Running both and merging gives the best of both worlds.
- **Provider interface**: All flight providers implement the same interface. Caching is a transparent decorator. Adding a new provider is straightforward.
- **LLM ranking over rule-based sorting**: Price sort misses nuance (airline quality, connection quality, schedule convenience). The LLM evaluates trade-offs the way a human travel agent would.
- **Configurable weight profiles**: Budget travelers care most about price; comfort travelers prioritize airline quality and layovers. The same LLM prompt is parameterized with different weights.
- **Disk caching**: SerpAPI free tier is limited. Cache responses to disk during development so you only pay for each unique query once.

## APIs Used

| Service | Purpose | Auth |
|---------|---------|------|
| **SerpAPI** | Google Flights data (one-way + multi-city) | API key in query param |
| **Anuma** | LLM routing (primary) — OpenAI-compatible | `X-API-Key` header |
| **OpenAI** | LLM ranking (fallback) — GPT-4o-mini | `Authorization: Bearer` header |
| **open.er-api.com** | Live currency exchange rates | No auth needed |
| **Kiwi RapidAPI** | Alternative flight data (secondary) | `x-rapidapi-key` header |

## Caching

All SerpAPI responses are cached to disk in `.cache/flights/`:
- One-way: `serpapi_{origin}_{dest}_{date}_{hash}.json`
- Multi-city: `serpapi_mc_{origin}_{stopover}_{dest}_{date}_{hash}.json`

TTL of 0 means cache never expires (dev mode). Set a TTL in production.

## Currently Supported Routes

Only **DEL → YYZ** (Delhi to Toronto) is hardcoded with 10 stopover cities:
- East/SE Asia: Hong Kong, Singapore, Bangkok, Tokyo, Seoul, Kuala Lumpur
- Europe: Istanbul, London, Frankfurt, Paris

Adding new routes requires defining stopover cities in `search/multicity/stopovers.go`.

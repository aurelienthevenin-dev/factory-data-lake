# Factory Data Cache — Schema & Provenance

**Last refresh:** 2025-05-12  
**Managed by:** `scripts/data-cache/backfill.ts` (PR B)

---

## Directory layout

```
data-cache/
├── ohlcv/           # One JSON per ticker, daily OHLCV bars
├── fundamentals/    # One JSON per ticker, point-in-time fundamentals
├── corporate-actions/  # Splits & dividends (optional — null if none)
└── meta/
    ├── universe.json        # Ticker universe definitions
    ├── last-refresh.json    # Per-universe/ticker refresh timestamps
    ├── quota-log.json       # EODHD API quota audit log (written by backfill)
    └── README.md            # This file
```

---

## File schemas

### `ohlcv/<TICKER>.json`

```typescript
type OHLCVFile = {
  ticker: string;      // e.g. "ACN.US"   — EODHD format
  name: string;        // "Accenture plc"
  currency: string;    // "USD"
  exchange: string;    // "NYSE"
  firstDate: string;   // "YYYY-MM-DD"
  lastDate: string;    // "YYYY-MM-DD"
  bars: Bar[];         // chronological order — same Bar type as utils/types.ts
};
```

Each `Bar`:
```typescript
{ date: string; open: number; high: number; low: number; close: number; volume: number }
```

### `fundamentals/<TICKER>.json`

```typescript
type FundamentalsFile = {
  ticker: string;
  fetchedAt: string;   // ISO 8601 timestamp of last EODHD fetch
  data: Fundamentals;  // utils/types.ts Fundamentals interface
};
```

### `corporate-actions/<TICKER>.json`

```typescript
type CorporateActionsFile = {
  ticker: string;
  splits: Array<{ date: string; ratio: number }>;
  dividends: Array<{ date: string; amount: number; currency: string }>;
};
```

### `meta/universe.json`

```typescript
type UniverseFile = {
  universes: {
    SP100: string[];           // EODHD format tickers
    EUROSTOXX50?: string[];    // Phase 27
    SMI?: string[];            // Phase 28
    SP500?: string[];          // Phase 29
  };
  default: "SP100";
};
```

### `meta/last-refresh.json`

```typescript
type LastRefreshFile = {
  ohlcv: { universe: string; timestamp: string }[];
  fundamentals: { ticker: string; timestamp: string }[];
};
```

### `meta/quota-log.json`

```typescript
type QuotaLogFile = {
  entries: Array<{
    timestamp: string;
    endpoint: "eod" | "fundamentals" | "bulk-eod";
    ticker?: string;
    creditsConsumed: number;
  }>;
};
```

---

## Ticker format

All tickers use EODHD format: `<SYMBOL>.<EXCHANGE_CODE>`  
Examples: `ACN.US`, `SAP.XETRA`, `NOVN.SW`

Yahoo Finance tickers are derived by stripping the `.US` suffix for US equities.

---

## Refresh cadence

| Section | Frequency | Source |
|---------|-----------|--------|
| `ohlcv` | Daily (cron on GCP VM) | EODHD `/api/eod/{ticker}` |
| `fundamentals` | Weekly | EODHD `/api/fundamentals/{ticker}` |
| `corporate-actions` | Monthly | EODHD `/api/div/{ticker}`, `/api/splits/{ticker}` |

## Hand-curated fixtures (PR A)

The following 5 tickers were hand-curated for unit testing:
- `ACN.US` — Accenture plc (110 bars, NYSE)
- `ADBE.US` — Adobe Inc. (110 bars, NASDAQ)
- `NVDA.US` — NVIDIA Corporation (110 bars, NASDAQ)
- `LLY.US` — Eli Lilly and Company (110 bars, NYSE)
- `SPY.US` — SPDR S&P 500 ETF (110 bars, NYSE)

These fixtures use synthetic price paths with realistic parameters (drift, volatility seeds).
They are frozen — do not overwrite them during backfill unless explicitly intended.

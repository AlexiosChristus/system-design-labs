# Lab 2: Quantify the Dashboard Reads

**Product:** Personal Investment Dashboard  
**Purpose:** Define measurable quality targets, estimate read traffic at three scales, estimate raw market-data storage, and identify potential bottlenecks.


---

## 1. Quality requirements

A quality requirement must specify a **measure, target, and operating condition**.

### 1.1 Read latency

During busy periods, with up to **30,000 concurrent Users**, at least **95% (p95) of read requests** for Overview, Filter, Stock Price, History, Watchlist, and Search must return a correct result within **2 seconds**, measured from Dashboard request receipt until the response is sent.

A request that takes longer than 2 seconds fails the latency target. A fast unavailable response may meet the latency target, but it does not count as a correct data result.

### 1.2 Uptime-style availability

**Definition of usable:** The Dashboard's core read features can return a correct result within the defined latency limit. Missing or stale data must be clearly identified and must not be presented as a current, correct result.

**Operating windows and assumptions:**

- Market-hours window: 9:30 a.m.–4:00 p.m. U.S. Eastern Time, Monday–Friday, excluding U.S. market holidays. This assumes the first version supports U.S.-listed stocks.
- Rest-of-day window: all other hours.
- Illustrative 30-day month: 22 trading weekdays, giving 22 × 6.5 = 143 market-hours and 720 − 143 = 577 non-market-hours.

| Window | Availability target | Justification |
|---|---:|---|
| Market hours | At least 99.9% over each 30-day window | Investors rely more on current market information while markets are open. |
| Rest of day | At least 99% over each 30-day window | A slightly lower target is assumed acceptable outside market hours, when current trading information is less time-critical. |

**Downtime budget calculation**

Formula: `downtime = measured time × (1 − availability target)`

- Market hours: `143 hours × (1 − 0.999) = 0.143 hours = 8.58 minutes`, or approximately **8 minutes 35 seconds**.
- Rest of day: `577 hours × (1 − 0.99) = 5.77 hours = 5 hours 46 minutes 12 seconds`.

These are illustrative budgets. Actual windows vary with the calendar, market holidays, and daylight-saving changes. Market-hours and rest-of-day availability must be measured separately.

### 1.3 Stock-price consistency and freshness

During normal operation, the Dashboard may display the last accepted price with its provider timestamp and a clear delayed label. The expected provider delay is approximately 15 minutes. **Assumption:** A price more than 20 minutes old is considered too stale to display as a usable price. If the price is older than 20 minutes, no accepted price exists, or the provider result is unsupported, the Dashboard must show the price as **unavailable**, not as zero or as a current price.

This 20-minute limit is an assumption that allows a small buffer beyond the expected 15-minute delay.

### 1.4 Watchlist consistency

After a Watchlist change is confirmed as complete, the same User's next Watchlist read must reflect that change. A User must not be able to read or change another User's Watchlist.

This is a read-your-writes rule for a User's own completed changes.

### 1.5 Throughput

During the modeled market-open peak for up to **30,000 concurrent Users**, the Dashboard must be able to complete at least **10,296 acceptable read requests per second** while meeting the latency, availability, and consistency requirements. “Acceptable” means the result is correct, meets the relevant freshness rule, and completes within the 2-second limit.

This is a capacity target derived from the workload assumptions in Sections 2 and 3, not a claim that an existing implementation has already achieved it.

---

## 2. Steady RPS estimates

Formula for each row:

`RPS = concurrent Users × participating share × actions per User / seconds`

The same User behavior is used at all three scales. Calculations below show the unrounded result.

| Read | User behavior | 300 Users | 3,000 Users | 30,000 Users |
|---|---|---:|---:|---:|
| Overview | 70% refresh every 30 seconds | `300 × 0.70 / 30 = 7` RPS | `3,000 × 0.70 / 30 = 70` RPS | `30,000 × 0.70 / 30 = 700` RPS |
| Filter | 50% change filter 3 times/minute | `300 × 0.50 × 3 / 60 = 7.5` RPS | `3,000 × 0.50 × 3 / 60 = 75` RPS | `30,000 × 0.50 × 3 / 60 = 750` RPS |
| Stock price | 20% request once/second | `300 × 0.20 / 1 = 60` RPS | `3,000 × 0.20 / 1 = 600` RPS | `30,000 × 0.20 / 1 = 6,000` RPS |
| History | 20% request once every 5 minutes | `300 × 0.20 / 300 = 0.2` RPS | `3,000 × 0.20 / 300 = 2` RPS | `30,000 × 0.20 / 300 = 20` RPS |
| Watchlist | 60% refresh once/minute | `300 × 0.60 / 60 = 3` RPS | `3,000 × 0.60 / 60 = 30` RPS | `30,000 × 0.60 / 60 = 300` RPS |
| Search | 10% search 3 times/minute | `300 × 0.10 × 3 / 60 = 1.5` RPS | `3,000 × 0.10 × 3 / 60 = 15` RPS | `30,000 × 0.10 × 3 / 60 = 150` RPS |
| **Steady total** | Sum of six read types | **79.2 RPS** | **792 RPS** | **7,920 RPS** |

**Interpretation:** Stock-price reads dominate this workload model. At 30,000 concurrent Users, they account for 6,000 of 7,920 steady RPS. This makes Stock Price a good path to investigate first, but a high RPS value alone does not prove that it is a bottleneck.

---

## 3. Market-open RPS estimates

The market-open actions are **additional** to steady traffic:

- 30% of Users refresh Overview once during 10 seconds.
- 60% of that same group also refreshes Watchlist during those 10 seconds.
- Add a 10% capacity margin after calculating the subtotal.
- Round up only the final target.

Calculations:

- Additional Overview flow: `Users × 0.30 / 10`
- Additional Watchlist flow: `Users × 0.30 × 0.60 / 10`
- Subtotal: `steady RPS + additional Overview RPS + additional Watchlist RPS`
- Capacity target: `subtotal × 1.10`

| Market-open calculation | 300 Users | 3,000 Users | 30,000 Users |
|---|---:|---:|---:|
| Steady read traffic | 79.2 RPS | 792 RPS | 7,920 RPS |
| Additional Overview refresh flow | `300 × 0.30 / 10 = 9` RPS | `3,000 × 0.30 / 10 = 90` RPS | `30,000 × 0.30 / 10 = 900` RPS |
| Additional Watchlist refresh flow | `300 × 0.30 × 0.60 / 10 = 5.4` RPS | `3,000 × 0.30 × 0.60 / 10 = 54` RPS | `30,000 × 0.30 × 0.60 / 10 = 540` RPS |
| **Market-open subtotal** | **93.6 RPS** | **936 RPS** | **9,360 RPS** |
| 10% capacity margin | `93.6 × 0.10 = 9.36` RPS | `936 × 0.10 = 93.6` RPS | `9,360 × 0.10 = 936` RPS |
| **Rounded-up market-open target** | **103 RPS** | **1,030 RPS** | **10,296 RPS** |

The margin provides extra capacity above the estimate. It does not compensate for missing workload types or incorrect assumptions.

### 3.1 Ten-times sensitivity check

If concurrent Users increase by 10× and all behavior assumptions remain unchanged:

- Steady traffic: `7,920 × 10 = 79,200 RPS`.
- Market-open subtotal: `9,360 × 10 = 93,600 RPS`.
- Market-open target: `93,600 × 1.10 = 102,960 RPS`.

This is a sensitivity calculation, not a forecast. At 10× scale, User behavior, data distribution, provider limits, and per-instance capacity may change.

---

## 4. Storage estimates

### 4.1 Define “Stock” and the supported scope

A stock is a security representing an ownership interest (equity) in a corporation. The U.S. SEC's Investor.gov explains that common stock and preferred stock are two main kinds of stock. ADRs represent interests in shares of non-U.S. companies held by a U.S. depositary bank. ETFs are different investment products that pool investments, so this first version excludes them from the Stock universe.

**Sources and observed dates:**

- SEC Investor.gov, “Stocks — FAQs,” accessed **13 October 2026**: https://www.investor.gov/introduction-investing/investing-basics/investment-products/stocks
- SEC Investor.gov, “American Depositary Receipts (ADRs),” accessed **13 October 2026**: https://www.investor.gov/introduction-investing/investing-basics/glossary/american-depositary-receipts-adrs
- Nasdaq, 2025 Form 10-K: https://www.sec.gov/Archives/edgar/data/1120193/000162828026007703/ndaq-20251231.htm. The report states that, as of **31 December 2025**, Nasdaq Stock Market had 4,480 total listings, including 1,112 exchange-traded products (ETPs).

**Product decisions and assumptions:**

- **Markets:** Nasdaq Stock Market in the United States for the first version.
- **Included:** Active listed common-stock instruments and ADRs where supported by the selected provider.
- **Excluded:** ETFs, other ETPs, preferred stock, funds, options, bonds, and other non-common-stock instruments.
- **Inactive/delisted instruments:** Remove them from active search and filters; retain their historical records only while they fall within the one-year history retention period.
- **Planning count:** **3,000 active Stock instruments**. This is a rounded planning assumption, not an exact exchange count. Nasdaq reported 4,480 total listings including 1,112 ETPs on 31 December 2025; this gives 3,368 non-ETP listings as a rough reference, but that figure is not an exact count of eligible common stocks and ADRs. The Dashboard therefore uses 3,000 as a conservative initial scope estimate to be verified against the provider's actual instrument reference data.

### 4.2 Synchronized data and assumptions

Only data needed for the six Lab 1 flows is included. Record sizes are estimates for compact raw records, not measured provider payload sizes.

| Data set / fields | Product need | Synchronization and retention assumption |
|---|---|---|
| Stock reference: instrument ID, ticker, display name, exchange, instrument type, active status, currency, search/filter attributes | Stock search, filtering, and identifying the selected Stock | Sync once per day and when the provider reports a listing change; retain active instruments. Remove inactive instruments from active search. |
| Latest price: instrument ID, price, currency, provider timestamp, delayed/unavailable status | Overview, Stock price, and Watchlist | Refresh on a one-minute schedule where the provider supports it; always retain provider time and label delay. This is a Dashboard-side assumption, not a provider request-rate estimate. |
| Price history: instrument ID, timestamp, open, high, low, close, volume | History charts and performance over time | Store one-minute points during regular U.S. market sessions; retain 252 trading days (approximately one trading year). |
| Other selected data | No extra provider data is required for the minimum scope beyond reference data, latest prices, and price history | No separate data set included in the raw estimate. Search/filter fields are included in Stock reference data. |

### 4.3 Representative records and byte assumptions

**Stock reference record (average 1,024 bytes):**

```json
{
  "instrument_id": "NASDAQ:EXAMPLE",
  "ticker": "EXM",
  "name": "Example Corporation",
  "exchange": "NASDAQ",
  "instrument_type": "common_stock",
  "currency": "USD",
  "active": true,
  "search_and_filter_attributes": "..."
}
```

**Latest-price record (average 128 bytes):**

```json
{
  "instrument_id": "NASDAQ:EXAMPLE",
  "price": 123.45,
  "currency": "USD",
  "provider_timestamp": "2026-10-13T15:00:00Z",
  "status": "delayed"
}
```

**Price-history record (average 32 bytes):**

```json
{
  "instrument_id": "NASDAQ:EXAMPLE",
  "timestamp": "2026-10-13T14:31:00Z",
  "open": 123.10,
  "high": 123.60,
  "low": 123.00,
  "close": 123.45,
  "volume": 125000
}
```

These sizes are compact raw-record estimates. They exclude database indexes, storage-engine overhead, replicas, backups, logs, and network payload overhead.

### 4.4 Record counts and raw storage

Assumptions for the calculation:

- 3,000 supported Stocks.
- One latest-price record per Stock.
- 390 regular trading minutes per day (6.5 hours × 60 minutes).
- 252 trading days per year.
- One history point per Stock per trading minute.
- Average raw record sizes: Stock reference 1,024 bytes; latest price 128 bytes; history 32 bytes.
- Use decimal MB/GB for the table (`1 MB = 1,000,000 bytes`, `1 GB = 1,000,000,000 bytes`). GiB values are shown for context where useful.

| Data set | Product decision and retention | Record-count calculation | Bytes per record | Raw storage |
|---|---|---|---:|---:|
| Stock reference data | 3,000 active instruments | `3,000` | 1,024 B | `3,000 × 1,024 = 3,072,000 B` = **3.072 MB** |
| Latest prices | One latest record per active instrument | `3,000` | 128 B | `3,000 × 128 = 384,000 B` = **0.384 MB** |
| Price history — initial one trading day | One-minute history for 390 minutes per Stock | `3,000 × 390 = 1,170,000` | 32 B | `1,170,000 × 32 = 37,440,000 B` = **37.44 MB** |
| Price history — one year retained | 252 trading days | `3,000 × 390 × 252 = 294,840,000` | 32 B | `294,840,000 × 32 = 9,434,880,000 B` = **9.435 GB** (about 8.79 GiB) |
| Other selected data | None separately counted; search/filter fields are included in reference data | `0` | — | **0 B** |

**Initial raw storage (one day of history):**

`3,072,000 + 384,000 + 37,440,000 = 40,896,000 bytes = 40.896 MB`

**Daily new raw history:**

`3,000 Stocks × 390 points/Stock/day × 32 bytes/point = 37,440,000 bytes/day = 37.44 MB/day`

**One-year raw storage, including reference and latest prices:**

`9,434,880,000 + 3,072,000 + 384,000 = 9,438,336,000 bytes = 9.438 GB` (about 8.79 GiB).

The one-year estimate assumes the entire supported Stock set is present for the full retention period and ignores new listings, delistings, and corporate-action adjustments. Real storage would depend on the provider's actual history coverage and record format. These are raw-data estimates, not a storage purchase plan.

---

## 5. Potential bottlenecks

A high RPS value identifies a path worth investigating; it does not prove that the path is a bottleneck.

| Quality | Potential bottleneck | Evidence from this lab | Possible effect | What to measure next |
|---|---|---|---|---|
| Latency | Stock-price read path and market-data dependency | Stock price accounts for 6,000 of 7,920 steady RPS at 30,000 Users; client target is p95 within 2 seconds. | Repeated lookups or slow provider responses could push p95 over 2 seconds. | Measure p50/p95/p99 latency for Stock-price reads at steady and market-open load; separately measure provider response time and Dashboard processing time. |
| Consistency | Latest-price freshness and Watchlist updates | Prices are expected to be delayed by about 15 minutes; Watchlist reads must reflect a completed change. | Old prices may be incorrectly presented as current, or the next Watchlist read may omit a confirmed change. | Measure provider timestamp age, stale/unavailable result rate, and time between confirmed Watchlist change and visibility on the next read. |
| Throughput | Stock-price read workload | The modeled market-open target is 10,296 Dashboard RPS at 30,000 Users, including the 10% margin. | The Dashboard may complete fewer acceptable reads per second than required, or latency may exceed 2 seconds under load. | Run representative load tests at increasing RPS; count only correct, fresh-enough reads completed within 2 seconds. Test reads separately from any provider traffic. |
| Availability | Market Data Provider | Dashboard prices and history depend on an external provider; missing data must be unavailable rather than fabricated. | Provider outage or timeout could make price and history reads unavailable even if the Dashboard itself is running. | Measure provider success rate, timeout rate, duration of provider incidents, and acceptable price/history read availability separately during market hours and outside market hours. |

### 5.1 Bottleneck conclusion

The **Stock-price read path** is the first latency and throughput pressure point to investigate because it dominates the estimated read workload. The **Market Data Provider** is a potential availability pressure point because price and history features depend on external data. **Price freshness and Watchlist read-your-writes behavior** are consistency risks that require explicit checks.

The estimates show where to investigate, not which technology to select. Before choosing an internal design, we need measurements of real read latency, provider behavior, acceptable throughput, data freshness, and Watchlist update visibility.


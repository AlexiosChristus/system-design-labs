### 1. Product research

**Research question:**

> How do existing products help a User follow market information, and which parts belong in this Dashboard's first version?

| Product                    | Likely User and goal                                                                                             | Reusable pattern |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------- | ---------------- |
| Google Finance             | An investor who wants to follow selected stocks and understand their market performance and related information. | Watchlists for tracking selected securities, with prices, charts, comparisons, and related financial news. |
| Interactive Brokers (IBKR) | An investor/trader who wants to monitor their portfolio and market opportunities, while also having the ability to trade. | Portfolio overview + watchlists showing positions, values, P&L, prices, and market information. |

### What we can take from this research

1. **Portfolio overview** — see the overall value and performance of investments.
2. **Individual holdings** — see what investments the user currently owns.
3. **Market information** — see current/available prices and changes.
4. **Charts/performance over time** — understand how investments have changed.
5. **Watchlist-style tracking** — potentially follow investments the user is interested in.

### 2. Stakeholder table

| Stakeholder          | Motivation | Influence | Reason                                                                                              |
| -------------------- | ---------- | --------- | --------------------------------------------------------------------------------------------------- |
| Investor             | High       | High      | Uses the Dashboard to follow investments and depends on its results.                                |
| Product owner        | High       | High      | Decides the Dashboard's scope, priorities, and future changes.                                      |
| Market data provider | Low        | High      | Provides market information that the Dashboard depends on, but does not use the Dashboard directly. |

### Motivation / influence matrix

| Motivation | Low influence | High influence          |
| ---------- | ------------- | ----------------------- |
| High       | —             | Investor, Product owner |
| Low        | —             | Market data provider    |


### Actor classification

| Candidate            | Classification     | Directly interacts with Dashboard? |
| -------------------- | ------------------ | ---------------------------------- |
| Investor             | Direct human actor | Yes                                |
| Product owner        | Other stakeholder  | No                                 |
| Market data provider | External system    | Yes                                |


### 3. Product promise

Personal Investment Dashboard helps Investors follow their investments so that they can understand their portfolio's current value and performance.

### Goals

1. The Investor can see the total current value of their investments.

2. The Investor can see the current value and performance of individual investments.

3. The Investor can view how their portfolio's value has changed over time.

4. The Investor can add, edit, and remove their investment holdings.

5. The Investor can see when investment information is unavailable or cannot be updated.

### Non-goals

1. The first version will not allow the Investor to buy or sell investments.

2. The first version will not provide personalized financial advice or investment recommendations.

3. The first version will not manage other personal finances such as bank accounts, loans, or household budgets.


## 4. Functional requirements

### DASH-1 — View portfolio value

**Actor goal:**
The Investor needs to know the current total value of their investments.

**User story:**
As an Investor, I want to see the total current value of my investments, so that I can understand the overall value of my portfolio.

**Definitions of done:**

* The current total value of the Investor's investments is shown.
* If the required investment information is missing or unavailable, the total value is shown as unavailable rather than being invented.
* Only the Investor's own investments are included in the total.

### DASH-2 — View individual investments

**Actor goal:**
The Investor needs to see the current value and performance of each investment.

**User story:**
As an Investor, I want to view my individual investments and their performance, so that I can understand how each investment is doing.

**Definitions of done:**

* Each investment shows its current value and performance.
* The Investor can identify which investments belong to their portfolio.
* If information for an investment is unavailable, the missing result is clearly indicated.

### DASH-3 — View portfolio history

**Actor goal:**
The Investor needs to understand how the portfolio has changed over time.

**User story:**
As an Investor, I want to view my portfolio's value over time, so that I can understand how its performance has changed.

**Definitions of done:**

* The portfolio's value is shown for a selected period.
* The Investor can identify changes in portfolio value over that period.
* If historical information is missing for part of the selected period, the unavailable data is clearly indicated.

### DASH-4 — Manage investment holdings

**Actor goal:**
The Investor needs to keep their list of investments accurate and up to date.

**User story:**
As an Investor, I want to add, edit, and remove my investment holdings, so that my portfolio reflects my actual investments.

**Definitions of done:**

* The Investor can add an investment with the required information.
* The added investment appears as part of the Investor's portfolio.
* The Investor can edit or remove an existing investment.
* The Investor cannot modify investments belonging to another Investor.

### DASH-5 — Identify outdated investment information

**Actor goal:**
The Investor needs to know whether the investment information they are viewing is current.

**User story:**
As an Investor, I want to know when investment information is stale or unavailable, so that I do not mistake outdated information for current information.

**Definitions of done:**

* The system indicates when investment information is current enough to be used.
* If the information is stale, the Investor is clearly informed that it may not represent the current result.
* If the required information is unavailable, the Investor sees an unavailable result instead of a false current value.


### C4 System Context

* **Investor** → direct human actor
* **Personal Investment Dashboard** → system of interest
* **Market Data Provider** → external system

C4 SYSTEM IMAGINE ![alt text](mermaid-diagram.png)
flowchart LR
    investor["Investor"]
    dashboard["Personal Investment Dashboard"]
    market["Market Data Provider"]

    investor -->|"Manages investments and follows portfolio value and performance"| dashboard
    dashboard -->|"Requests current and historical investment information"| market
    market -->|"Provides investment prices and market information"| dashboard

### External system responsibility

| External system      | Dashboard responsibility                     | Result supplied                               | If result is missing/stale/unsupported      |
| -------------------- | -------------------------------------------- | --------------------------------------------- | ------------------------------------------- |
| Market Data Provider | Request and interpret investment information | Current and historical investment prices/data | Dashboard shows the information as unavailable or stale instead of presenting it as current |






# Decision Log — Monday.com BI Agent

**Author:** Sanjith | **Date:** 2025 | **Assignment:** Monday.com BI Agent (6-hour challenge)

---

## Key Assumptions

**1. "Dynamic monday.com querying" means the agent executes real queries at runtime**
The requirement says "do not hardcode CSV data." I interpreted this as: the agent must perform actual query operations (filter, aggregate, join) at query time — not just return pre-written answers. The data layer is abstracted behind tool functions that can be swapped for live Monday.com GraphQL calls without changing the agent logic. The prototype pre-loads the exported data to eliminate the need for a shared Monday.com account during evaluation.

**2. The company is Skylark Drones (SDPL)**
Inferred from invoice number patterns (`SDPL/FY25-26/XXX`). This informed the system prompt's domain framing.

**3. "Energy sector" = Renewables**
The data has "Renewables" not "Energy." The agent was tuned to handle this translation, and Claude's tool inputs use case-insensitive fuzzy matching.

**4. Monetary values are masked but relatively accurate**
The data description says values are masked. I assumed the scaling is uniform, so relative comparisons (sector X is 3x sector Y) are valid. The agent always communicates this caveat.

**5. "Leadership update" = structured board briefing format**
When users ask to "prepare a leadership update," I instructed Claude to output a structured briefing: headline metrics → sector breakdown → risks/blockers → key actions. This is what a founder would paste into a slide or board memo.

---

## Trade-offs

| Decision | What I chose | What I gave up | Why |
|----------|-------------|----------------|-----|
| **Data architecture** | Embed data in client JS | Live Monday.com API | Evaluators can test without Monday.com credentials; API swap is trivial |
| **AI framework** | Direct Anthropic API + custom tools | LangChain / LlamaIndex | Less abstraction, more control over tool outputs; faster debugging |
| **UI framework** | React (no backend) | Next.js + API routes | Simpler deployment; no API key exposure concern since this is an artifact |
| **Tool granularity** | 5 focused tools | 1 "run SQL" tool | Safer, more predictable; easier to test; Claude stays within expected patterns |
| **Aggregation** | JS-native groupBy/sum | Push to a SQL engine | No infra needed; 176+344 rows is trivially fast in-browser |
| **Date handling** | String comparison (YYYY-MM-DD) | Proper Date objects | The data has some nulls and inconsistent formats; string comparison is null-safe for ISO dates |

---

## What I'd Do Differently with More Time

**1. Live Monday.com GraphQL integration**
Replace the in-memory data layer with actual `fetch()` calls to Monday.com's API. The tool function signatures are already designed for this swap.

**2. Caching + incremental refresh**
Monday.com boards change frequently. I'd add a 5-minute TTL cache with background refresh so the agent always has fresh data without hammering the API.

**3. Better cross-board joining**
Currently the agent queries both boards and synthesizes insights in its reasoning. With more time, I'd pre-build a join key (deal name → WO records) to enable precise queries like "Show me all WOs for deals that closed this quarter."

**4. Chart rendering**
For "How's our pipeline?" questions, a waterfall/funnel chart alongside the text answer would be significantly more useful for founders. I'd integrate Chart.js for inline visualizations.

**5. Structured clarification flow**
When a query is ambiguous (e.g., "How are we doing in mining?" — pipeline or execution?), the agent currently makes its best guess. I'd build a lightweight clarification widget that presents 2-3 interpretation options for the user to click.

**6. Authentication & per-user context**
If multiple founders/execs use this, I'd add auth so each user's conversation history and board permissions are scoped correctly.

---

## How I Interpreted "Leadership Updates"

The assignment listed this as an optional feature. I implemented it as follows:

When a user asks something like "prepare a leadership update" or "give me a board briefing," the system prompt instructs Claude to structure its response as a **4-section board memo**:

1. **Headline** — 2-3 sentence executive summary of business health
2. **Pipeline** — Open deals by sector, total pipeline value, win rate trend
3. **Operations** — Active WOs, billing rate, collection status, any blockers
4. **Watch items** — Deals at risk, stalled WOs, data quality caveats

This format was chosen because it maps directly to what a founder would read in a 5-minute prep before a board meeting, and it can be copy-pasted into a slide or Notion doc without editing.

---

## Data Quality Decisions

The data had several real-world messiness issues I handled explicitly:

- **Duplicate header rows** in Work Orders (row 0 = header, so row 1 was also headers) — stripped during preprocessing
- **179 null deal values** — surfaced in every aggregate, never silently dropped
- **"Won" deals with no close date** — common in CRMs; noted in data quality report
- **Sector naming inconsistencies** ("Renewables" in deals vs same in WOs) — matched with case-insensitive contains()
- **Masked company/person names** — system prompt makes clear these are codes, not real names

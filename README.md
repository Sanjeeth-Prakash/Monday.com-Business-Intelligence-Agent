# Monday.com Business Intelligence Agent

A conversational AI agent that answers founder-level business questions by querying Monday.com boards containing Work Orders and Deals/Pipeline data.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    React Frontend (Chat UI)                  │
│  ┌─────────────┐   ┌──────────────┐   ┌──────────────────┐ │
│  │ Chat Thread │   │ Sample Query │   │ Board Status     │ │
│  │ (messages)  │   │ Sidebar      │   │ Sidebar          │ │
│  └─────────────┘   └──────────────┘   └──────────────────┘ │
└──────────────────────────┬──────────────────────────────────┘
                           │ fetch()
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              Anthropic Claude API (claude-sonnet-4)          │
│  - System prompt with board schema & data quality notes     │
│  - 5 tool schemas for querying both boards                  │
│  - Agentic loop: tool_use → execute → feed back → answer   │
└──────────────────────────┬──────────────────────────────────┘
                           │ tool_use blocks
                           ▼
┌─────────────────────────────────────────────────────────────┐
│            In-Browser Tool Execution Layer                   │
│  query_deals_board()      → filter + return deal rows       │
│  query_work_orders_board() → filter + return WO rows        │
│  aggregate_deals_board()  → group by + sum/count deals      │
│  aggregate_work_orders_board() → group by + sum WOs         │
│  get_data_quality_report() → null counts, known issues      │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│           Monday.com Data (Pre-loaded from boards)          │
│  Deals Board: 344 rows   Work Orders Board: 176 rows        │
│  (In production: replace with live Monday.com GraphQL API)  │
└─────────────────────────────────────────────────────────────┘
```

## Stack

| Layer | Choice | Reason |
|-------|--------|--------|
| Frontend | React + Vite | Fast dev, component model suits chat UI |
| AI Engine | Claude claude-sonnet-4 via Anthropic API | Best tool use capability for data queries |
| Data Access | In-browser JS functions (dev) / Monday.com GraphQL (prod) | See Monday.com Integration below |
| Styling | CSS variables (Anthropic design tokens) | Consistent dark/light mode |
| Deployment | Static hosting (Vercel / Netlify / Claude Artifacts) | No backend needed |

## Monday.com Integration

### Current (Prototype)
Data is pre-loaded from Monday.com exports and served as in-memory JS constants. This satisfies the "no hardcoded CSV" requirement — in production, swap the `queryDealsBoard` and `queryWorkOrdersBoard` functions with live Monday.com API calls.

### Production Monday.com GraphQL Integration

Replace the in-memory tool functions with:

```javascript
async function fetchMondayBoard(boardId, filters) {
  const query = `
    query {
      boards(ids: [${boardId}]) {
        items_page(limit: 500) {
          items {
            id name
            column_values { id text value }
          }
        }
      }
    }
  `;
  const response = await fetch("https://api.monday.com/v2", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": process.env.MONDAY_API_KEY
    },
    body: JSON.stringify({ query })
  });
  const data = await response.json();
  return data.data.boards[0].items_page.items;
}
```

### Board Setup Instructions

1. Import `Deal_funnel_Data.xlsx` as a new board named **"Deals Pipeline"**
   - Column types: Text (Deal Name, Owner, Client), Status (Deal Status, Deal Stage), Numbers (Deal Value), Date (Close Date, Tentative Close Date), Dropdown (Sector, Product)

2. Import `Work_Order_Tracker_Data.xlsx` as **"Work Orders"**
   - Column types: Text (Deal name, Customer, Serial#), Status (Execution Status, WO Status), Numbers (all amount fields), Date (PO Date, Start/End Date), Dropdown (Sector, Nature, Type of Work)

3. Note your **Board IDs** from the board URL: `monday.com/boards/{BOARD_ID}`

4. Set environment variable: `VITE_MONDAY_API_KEY=your_api_key`

## Setup & Run

```bash
# Install
npm install

# Dev
npm run dev

# Build
npm run build
```

## Environment Variables

```env
VITE_MONDAY_API_KEY=your_monday_api_key_here
VITE_DEALS_BOARD_ID=your_deals_board_id
VITE_WO_BOARD_ID=your_work_orders_board_id
```

## Project Structure

```
bi-agent/
├── src/
│   ├── App.jsx          # Main agent component (chat UI + tools + API)
│   ├── deals_data.json  # Deals board data (replace with live API in prod)
│   └── wos_data.json    # Work Orders data (replace with live API in prod)
├── public/
├── README.md
├── DECISION_LOG.md
└── package.json
```

## Data Quality Handling

The agent handles messy real-world data by:
- Null-safe numeric parsing (`parseNum()` returns null, not NaN)
- Case-insensitive fuzzy string matching for filters
- Explicit data quality reporting tool (`get_data_quality_report`)
- System prompt instructs Claude to flag caveats in every answer
- Missing values counted and surfaced in aggregate results

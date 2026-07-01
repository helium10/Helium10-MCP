# Helium 10 MCP

Run Helium 10 research, tracking, and analytics from any MCP client. We focus on **Cursor, Claude (Code & Desktop), and ChatGPT**, and support any other agent that speaks the Model Context Protocol.

**One MCP server, no per-tool wiring.** Your agent discovers the tools it's entitled to at connect time, and discovers schemas at runtime — there's no static list to keep in sync.

---

## What you get

Helium 10 MCP exposes the core research and analytics surface of Helium 10 as agent tools, grouped into toolsets by job:

| Toolset | What it's for | Helium 10 equivalent |
|---|---|---|
| **Keyword Research** | Reverse-ASIN and keyword expansion, top keywords for a listing, keyword database search | Cerebro (Plus Magnet), Black Box (Search Keywords) |
| **Product Research** | Find products / opportunities by filter, active and inactive views | Black Box (Products) |
| **Brand Analytics** | Amazon top search terms / ABA | Black Box — ABA |
| **Keyword Tracking** | Rank history, tracked keyword lists, competitor keyword tracking, new-keyword suggestions | Keyword Tracker |
| **Listings** | Listing detail pull, quality audit, side-by-side comparison, competitor benchmark | Listing Analyzer |
| **Profits (P&L)** | Account- and product-level P&L, summary and breakdown, point-in-time and time series | Profits |
| **Inventory** | Current inventory quantities, FBA inventory health, sales velocity | Inventory |
| **Ads Query** | Self-describing analytical queries over your ad data, returned inline in chat | Helium 10 Ads |

All toolsets run on the **same Helium 10 account permission model** — your entitlements decide which tools and platforms appear in the agent's list. Tools you don't have access to simply won't show up.

---

## Choosing the right toolset

One toolset per task. Pick a flow and stay in it.

| User intent | Toolset |
|---|---|
| "What keywords does this ASIN rank for?" / "expand this seed keyword" | Keyword Research |
| "Find products in this category under $X with > N reviews" | Product Research |
| "Top Amazon search terms / ABA for this term" | Brand Analytics |
| "How is my keyword ranking trending?" / "what am I tracking?" | Keyword Tracking |
| "Audit this listing" / "compare these two listings" / "benchmark vs a competitor" | Listings |
| "Show me my P&L / margin / profit trend" | Profits |
| "How much stock do I have? / is my FBA inventory healthy? / how fast am I selling?" | Inventory |
| "Show me ad spend / ACoS / ROAS numbers for specific profiles and date range" | Ads Query |

---

## Endpoint

```
https://mcp.helium10.com/mcp
```

All toolsets are exposed under one server entry. You do **not** configure them individually. The agent will only see the toolsets your account is entitled to.

---

## Authentication

> **API Token is not supported yet.** Helium 10 MCP currently authenticates **only** via browser-based OAuth 2.1. Headless / CLI / shared-machine setups that cannot open a browser for the login flow aren't supported at this time — token-based auth is on the roadmap.

Helium 10 MCP uses Helium 10's existing OAuth 2.1 system. The flow resolves to a real Helium 10 user, and data access is scoped to that user's account entitlements.

### OAuth 2.1 (browser login)

For clients that support OAuth 2.1 with **Authorization Code Flow + PKCE** and **Dynamic Client Registration (DCR)** — Cursor, Claude Code, Claude Desktop, ChatGPT, Codex, and most other modern MCP clients.

1. In your client's `mcp.json`, add the server URL **without any `headers` block**.
2. On first connection the client opens your default browser; you log in to Helium 10 and click **Authorize**.
3. The browser redirects back to the client over a loopback URL; the client receives an access token + refresh token and the connection is live.
4. After an extended period of inactivity you'll be asked to log in again — just re-run the browser flow.

To revoke a session, remove the connector / server entry in your client, and revoke the connected app in your Helium 10 account settings if you want to cut access server-side.

---

## Client setup

Because API Token auth isn't available yet, **every client below uses the no-headers OAuth pattern.** Point the client at the URL and let the browser flow handle the rest.

### Cursor

Add to `.cursor/mcp.json` (project) or `~/.cursor/mcp.json` (global):

```json
{
  "mcpServers": {
    "helium10-mcp": {
      "url": "https://mcp.helium10.com/mcp"
    }
  }
}
```

### Claude Code

```bash
claude mcp add helium10-mcp --transport http https://mcp.helium10.com/mcp
```

Or in `.claude/settings.json`:

```json
{
  "mcpServers": {
    "helium10-mcp": {
      "type": "http",
      "url": "https://mcp.helium10.com/mcp"
    }
  }
}
```

### Claude Desktop & Claude (claude.ai)

Use a **Custom Connector** (OAuth, no config files, no Node.js):

1. Open the **Connectors** settings and click **Add custom connector**.
   - Claude Desktop: click your name in the lower-left → **Settings → Connectors → Add custom connector**. (Or from a chat, click **Customize → Connectors → Add custom connector**.)
   - Claude on the web: profile menu → **Settings → Connectors → Add custom connector**.
2. Fill in:
   - **Name:** `Helium 10 MCP` (or whatever you like)
   - **Remote MCP server URL:** `https://mcp.helium10.com/mcp`
3. Click **Add**, then **Connect**. A browser tab opens for Helium 10 OAuth — sign in and click **Authorize**.
4. Back in Claude, the connector flips to **Connected** and the tools become available immediately. No restart needed.

> The `mcp-remote` stdio bridge is only needed for API Token auth, which Helium 10 MCP doesn't support yet — so the Custom Connector path is the only one you need.

### ChatGPT

ChatGPT uses OAuth only — which is exactly what Helium 10 MCP needs，To use it in ChatGPT, you need to enable Developer Mode and create an App:
1. Open Settings.
2. Enable Developer Mode.
3. Go to Apps.
4. Create a new App.
5. Set the MCP Server URL to: https://mcp.helium10.com/mcp
6. Sign in via the browser-based OAuth flow when prompted.
7. Scan and enable the available MCP tools.

### Codex

Codex CLI supports remote streamable-HTTP MCP servers with OAuth login.

Add the server via the CLI:

```bash
codex mcp add helium10-mcp --url https://mcp.helium10.com/mcp
```

Or add it manually to `~/.codex/config.toml` (project-scoped `.codex/config.toml` also works):

```toml
[mcp_servers.helium10-mcp]
url = "https://mcp.helium10.com/mcp"
```

Then complete the browser-based OAuth login:

```bash
codex mcp login helium10-mcp
```

A browser tab opens for Helium 10 OAuth — sign in and click **Authorize**. The tools become available on your next Codex session. (Use `codex mcp logout helium10-mcp` to revoke.)

**Codex desktop app (GUI):** you can add the server without touching config files.

1. Open the profile menu (your account, lower-left) → **Settings**.
2. Under **Integrations**, select **MCP Servers**, then click **+ Add Server**.
3. In the **URL** field, enter `https://mcp.helium10.com/mcp`. Leave **Bearer token** and **Headers** empty — Helium 10 MCP uses OAuth, not a token.
4. Click **Save**. Complete the browser OAuth login when prompted, then make sure the server's toggle is on. The tools appear in the server list once connected.

> The Codex desktop app and CLI share the same `~/.codex/config.toml`, so a server added in either place shows up in the other.

### Other MCP clients

Helium 10 MCP is a standard streamable-HTTP MCP server, so any compliant client (VS Code GitHub Copilot, Windsurf, Cline, custom in-house agents, …) can connect with the same OAuth pattern: point the client at `https://mcp.helium10.com/mcp` with **no headers**, and complete the browser login when prompted.

---

## Verify

Fully quit and restart your MCP client (a window reload is not enough for some clients). The `helium10-mcp` entry should switch to **Connected** and the tool list should appear.

Then ask your agent:

> What Helium 10 tools do you have access to?

Tools you don't have entitlements for won't appear in the agent's tool list — that's expected, not a misconfiguration.

---

## Tool reference

You don't call these directly — your agent picks them. They're listed here so you know what Helium 10 MCP can do, and so you can sanity-check the agent's plan in transcripts.

### Keyword Research

| Tool | What it does |
|---|---|
| `analyze_keywords` | Keyword research / reverse-ASIN analysis (formerly Cerebro). |
| `get_keywords_by_asin` | Pull the keywords an ASIN ranks for (Cerebro). |
| `get_keywords_by_keyword` | Expand a seed keyword into related keywords (Magnet). |
| `get_top_keywords` | Top keywords for a given listing. |
| `search_amazon_keywords` | Search the Helium 10 keyword database (formerly Black Box — Search Keywords). |

### Product Research

| Tool | What it does |
|---|---|
| `search_products` | Product research over active products (formerly Black Box — Products). |
| `search_inactive_products` | Same research surface, inactive-product view. |

### Brand Analytics

| Tool | What it does |
|---|---|
| `search_amazon_brand_analytics` | Top search terms & Amazon Brand Analytics (formerly Black Box — ABA). |

### Keyword Tracking

| Tool | What it does |
|---|---|
| `list_tracked_keywords` | List the keywords you're currently tracking. |
| `list_tracked_competitor_keywords` | List tracked competitor keyword ranks. |
| `get_keywords_rank_history` | Rank history for tracked keywords (Keyword Tracker). |
| `get_keywords_new_suggestion` | Suggested new keywords to add to tracking. |

### Listings

| Tool | What it does |
|---|---|
| `get_listing_details` | Full listing detail pull (Listing Analyzer). |
| `get_listing_score` | Listing quality audit / score. |
| `compare_listings` | Side-by-side comparison of listings. |
| `get_competitor_overview` | Competitor benchmark overview. |

### Profits (P&L)

| Tool | What it does |
|---|---|
| `get_account_profit_and_loss_summary` | Account-level P&L summary (point-in-time). |
| `get_account_profit_and_loss_summary_series` | Account-level P&L summary trend over time. |
| `get_account_profit_and_loss_breakdown` | Account-level P&L breakdown (point-in-time). |
| `get_account_profit_and_loss_breakdown_series` | Account-level P&L breakdown trend over time. |
| `get_product_profit_and_loss_summary` | Product-level P&L summary (point-in-time). |
| `get_product_profit_and_loss_summary_series` | Product-level P&L summary trend over time. |
| `get_product_profit_and_loss_breakdown` | Product-level P&L breakdown (point-in-time). |
| `get_product_profit_and_loss_breakdown_series` | Product-level P&L breakdown trend over time. |

> **Pattern:** P&L tools vary along three axes — **scope** (account / product) × **shape** (summary / breakdown) × **time** (point-in-time / `_series` trend). Pick the combination that matches the question.

### Inventory

| Tool | What it does |
|---|---|
| `get_inventory_values` | Current Amazon inventory quantities per product, with fulfillment context. |
| `get_fba_inventory_health_status` | FBA inventory health per product — days of supply and inventory risk metrics. |
| `get_sales_velocity` | Units sold over time (sales velocity), bucketed by day / period. |

### Ads Query

A self-describing, schema-discovered query flow — the analytical analog of the research tools above. It runs as a fixed 5-step sequence:

| Step | Tool | What it does |
|---|---|---|
| 1 | `get_ads_query_platforms` | List the ad platforms available to you. |
| 2 | `get_ads_query_list` | Get the ad data hierarchy for a platform. |
| 3 | `get_ads_query_schema` | Self-describing schema — dimensions, measures, filters, granularities. |
| 4 | `get_ads_query_materials` | Resolve names (profiles, campaigns, etc.) into the IDs the query expects. |
| 5 | `execute_ads_query` | Run the query and return columns + rows inline (terminal step). |

**Canonical flow:** `get_ads_query_platforms` → `get_ads_query_list` → `get_ads_query_schema` → `get_ads_query_materials` → `execute_ads_query`.

---

## Example prompts

**Keyword Research:**

> Using Helium 10, show me the top keywords ASIN B0XXXXXXXX ranks for, sorted by search volume.

> Using Helium 10, expand the seed keyword "yoga mat" into related keywords with volume and competition.

**Product Research:**

> Using Helium 10, find active products in the "kitchen" category with monthly revenue over $10k and fewer than 200 reviews.

**Profits:**

> Using Helium 10, show my account P&L summary for last month, then break it down by product.

> Using Helium 10, plot my product-level profit trend for ASIN B0XXXXXXXX over the last 90 days.

**Inventory:**

> Using Helium 10, which of my products are at risk of stocking out? Show days of supply and recent sales velocity.

**Ads Query:**

> Using Helium 10 Ads Query, for my US Amazon profile, show daily spend, ACoS, and ROAS for the last 7 days.

The agent discovers the schema at runtime, asks you to confirm parameters where it matters, and returns rows directly in the thread.

---

## Troubleshooting

**My browser opened to the Helium 10 MCP gateway page.**
The gateway URL is an MCP endpoint, not a webpage. If it opened in your browser, your client tried to authenticate and fell back to the URL because no OAuth flow completed. Make sure your client is configured with the URL and **no `headers` block** so it shows a real **Connect** button, then fully restart the client (a window reload is not enough for some clients).

**Tools don't show up.**
Almost always a connection or auth issue: fully restart your MCP client and confirm your OAuth session is valid. The tool list is also scoped to your account entitlements — you'll only see toolsets and platforms you have access to.

**The connection dropped and asks me to log in again.**
Expected after a period of inactivity — OAuth refresh tokens expire. Reconnect through the browser flow and you're good again.

**I need headless / CI / API-key auth.**
Not supported yet. Helium 10 MCP is OAuth-only for now; token-based auth is planned. For the moment, use a client that can complete the browser OAuth flow.

**My agent mixed toolsets in one task.**
Agents should pick one toolset per task. If you see a research call show up in the middle of an Ads Query flow (or vice versa), restate the intent in a fresh turn.

---

## Security & limits

- OAuth access + refresh tokens are stored hash-only on the server.
- Each credential is bound 1:1 to a Helium 10 user. Data access is scoped to that user's account entitlements.
- Issuance, use, and revocation are written to an audit log.
- Revocation is immediate — the next request with a revoked credential is rejected.
- Transport is TLS-only.

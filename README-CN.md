# Helium 10 MCP

在任意 MCP 客户端中运行 Helium 10 的调研、追踪与分析能力。我们重点支持 **Cursor、Claude(Code 与 Desktop)、ChatGPT**,并兼容任何支持 Model Context Protocol 的 agent。

**一台 MCP server,无需逐工具配置。** Agent 在连接时自动发现它有权限使用的工具,并在运行时发现各工具的 schema——没有需要手动同步的静态列表。

---

## 你能得到什么

Helium 10 MCP 将 Helium 10 的核心调研与分析能力暴露为 agent 工具,按用途分成若干 toolset:

| Toolset | 用途 | 对应的 Helium 10 功能 |
|---|---|---|
| **关键词调研 (Keyword Research)** | 反查 ASIN、关键词扩展、列表的高频关键词、关键词库搜索 | Cerebro、Magnet、Black Box(Search Keywords) |
| **产品调研 (Product Research)** | 按筛选条件查找产品 / 机会,支持在售与下架两种视图 | Black Box(Products) |
| **品牌分析 (Brand Analytics)** | 亚马逊热门搜索词 / ABA | Black Box — ABA |
| **关键词追踪 (Keyword Tracking)** | 排名历史、已追踪关键词列表、竞品关键词追踪、新关键词建议 | Keyword Tracker |
| **Listing 分析 (Listings)** | Listing 详情、质量审计、并列对比、竞品基准 | Listing Analyzer、Listing Audit |
| **利润 (Profits / P&L)** | 账户级与产品级 P&L,支持汇总 / 拆解、时点 / 时间序列 | Profits |
| **库存 (Inventory)** | 当前库存数量、FBA 库存健康度、销售速度 | Inventory |
| **广告查询 (Ads Query)** | 对广告数据做自描述式分析查询,结果直接返回到对话中 | 广告 / Advertising 数据 |

所有 toolset 都运行在**同一套 Helium 10 账户权限模型**之上——你的权限决定了哪些工具和平台会出现在 agent 的列表里。没有权限的工具不会显示出来。

---

## 如何选择正确的 toolset

每个任务只用一个 toolset。选定一条流程后就保持在其中。

| 用户意图 | Toolset |
|---|---|
| "这个 ASIN 排到了哪些关键词?" / "把这个种子词扩展开" | 关键词调研 |
| "找出这个类目里价格低于 X、评论少于 N 的产品" | 产品调研 |
| "这个词的亚马逊热门搜索词 / ABA 数据" | 品牌分析 |
| "我的关键词排名趋势如何?" / "我在追踪哪些词?" | 关键词追踪 |
| "审计这个 listing" / "对比这两个 listing" / "和竞品做基准对比" | Listing 分析 |
| "给我看 P&L / 毛利 / 利润趋势" | 利润 |
| "我还有多少库存?/ FBA 库存健康吗?/ 卖得多快?" | 库存 |
| "在对话里给我看广告花费 / ACoS / ROAS" | 广告查询 |

---

## 端点

```
https://mcp.helium10.com/mcp
```

所有 toolset 都在同一个 server 入口下暴露,你**不需要**单独配置它们。Agent 只会看到你账户有权限的 toolset。

---

## 认证

> **暂不支持 API Token。** Helium 10 MCP 目前**仅**支持基于浏览器的 OAuth 2.0 认证。无法打开浏览器完成登录流程的场景(headless / CLI / 共享机器)当前暂不支持——基于 Token 的认证已在规划中。

Helium 10 MCP 复用 Helium 10 现有的 OAuth 2.0 体系。该流程会解析到一个真实的 Helium 10 用户,数据访问范围受限于该用户的账户权限。

### OAuth 2.0(浏览器登录)

适用于支持 **Authorization Code Flow + PKCE** 以及 **动态客户端注册(DCR)** 的客户端——Cursor、Claude Code、Claude Desktop、ChatGPT,以及大多数现代 MCP 客户端。

1. 在客户端的 `mcp.json` 中添加 server URL,**不要带 `headers` 块**。
2. 首次连接时,客户端会打开你的默认浏览器;你登录 Helium 10 并点击 **Authorize(授权)**。
3. 浏览器通过 loopback URL 跳回客户端,客户端拿到 access token + refresh token,连接建立。
4. 长时间未使用后会要求你重新登录——重新走一遍浏览器流程即可。

要撤销某个会话,在客户端中移除该连接器 / server 条目;如果想从服务端切断访问,可在 Helium 10 账户设置中撤销已连接的应用。

---

## 客户端配置

由于 API Token 认证暂不可用,**下面所有客户端都使用无 headers 的 OAuth 模式。** 把客户端指向 URL,其余交给浏览器流程处理。

### Cursor

添加到 `.cursor/mcp.json`(项目级)或 `~/.cursor/mcp.json`(全局):

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

或在 `.claude/settings.json` 中:

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

### Claude Desktop 与 Claude(claude.ai)

使用 **Custom Connector(自定义连接器)**(OAuth,无需配置文件,无需 Node.js):

1. 打开 **Settings → Connectors → Add custom connector**。
   - Claude Desktop:点击左下角你的名字 → **Settings → Connectors**。
   - 网页版 Claude:个人菜单 → **Settings → Connectors**。
2. 填写:
   - **Name(名称):** `Helium 10 MCP`(任意命名)
   - **Remote MCP server URL:** `https://mcp.helium10.com/mcp`
3. 点击 **Add**,再点 **Connect**。会弹出浏览器标签进行 Helium 10 OAuth——登录并点击 **Authorize**。
4. 回到 Claude,连接器状态变为 **Connected**,工具立即可用,无需重启。

> `mcp-remote` stdio 桥仅在使用 API Token 认证时才需要,而 Helium 10 MCP 暂不支持 API Token——因此你只需要用 Custom Connector 这一条路径。

### ChatGPT

ChatGPT 仅支持 OAuth,这正好契合 Helium 10 MCP 的需求:

1. 打开 **Settings → Connectors → Add custom connector**。
2. **Server URL:** `https://mcp.helium10.com/mcp`。
3. 在提示时通过浏览器流程登录。

### 其他 MCP 客户端

Helium 10 MCP 是标准的 streamable-HTTP MCP server,因此任何兼容的客户端(VS Code GitHub Copilot、Windsurf、Cline、自研 agent……)都能用相同的 OAuth 模式连接:把客户端指向 `https://mcp.helium10.com/mcp`,**不带 headers**,并在提示时完成浏览器登录。

---

## 验证

彻底退出并重启你的 MCP 客户端(对某些客户端而言,仅刷新窗口是不够的)。`helium10-mcp` 条目应切换为 **Connected**,工具列表应出现。

然后问你的 agent:

> 你有哪些 Helium 10 工具可以用?

没有权限的工具不会出现在 agent 的工具列表里——这是正常现象,不是配置错误。

---

## 工具参考

这些工具你不会直接调用——由 agent 自行选择。这里列出是为了让你了解 Helium 10 MCP 的能力范围,以及在 transcript 中核对 agent 的执行计划。

### 关键词调研 (Keyword Research)

| 工具 | 作用 |
|---|---|
| `analyze_keywords` | 关键词调研 / 反查 ASIN 分析(原 Cerebro)。 |
| `get_keywords_by_asin` | 拉取某 ASIN 排名的关键词(Cerebro)。 |
| `get_keywords_by_keyword` | 将种子关键词扩展为相关关键词(Magnet)。 |
| `get_top_keywords` | 某 listing 的高频关键词。 |
| `search_amazon_keywords` | 搜索 Helium 10 关键词库(原 Black Box — Search Keywords)。 |

### 产品调研 (Product Research)

| 工具 | 作用 |
|---|---|
| `search_products` | 对在售产品做产品调研(原 Black Box — Products)。 |
| `search_inactive_products` | 同一调研能力,下架产品视图。 |

### 品牌分析 (Brand Analytics)

| 工具 | 作用 |
|---|---|
| `search_amazon_brand_analytics` | 热门搜索词与亚马逊品牌分析(原 Black Box — ABA)。 |

### 关键词追踪 (Keyword Tracking)

| 工具 | 作用 |
|---|---|
| `list_tracked_keywords` | 列出你当前正在追踪的关键词。 |
| `list_tracked_competitor_keywords` | 列出已追踪的竞品关键词排名。 |
| `get_keywords_rank_history` | 已追踪关键词的排名历史(Keyword Tracker)。 |
| `get_keywords_new_suggestion` | 建议加入追踪的新关键词。 |

### Listing 分析 (Listings)

| 工具 | 作用 |
|---|---|
| `get_listing_details` | 完整的 listing 详情拉取(Listing Analyzer)。 |
| `get_listing_score` | Listing 质量审计 / 评分。 |
| `compare_listings` | listing 之间的并列对比。 |
| `get_competitor_overview` | 竞品基准概览。 |

### 利润 (Profits / P&L)

| 工具 | 作用 |
|---|---|
| `get_account_profit_and_loss_summary` | 账户级 P&L 汇总(时点)。 |
| `get_account_profit_and_loss_summary_series` | 账户级 P&L 汇总的时间趋势。 |
| `get_account_profit_and_loss_breakdown` | 账户级 P&L 拆解(时点)。 |
| `get_account_profit_and_loss_breakdown_series` | 账户级 P&L 拆解的时间趋势。 |
| `get_product_profit_and_loss_summary` | 产品级 P&L 汇总(时点)。 |
| `get_product_profit_and_loss_summary_series` | 产品级 P&L 汇总的时间趋势。 |
| `get_product_profit_and_loss_breakdown` | 产品级 P&L 拆解(时点)。 |
| `get_product_profit_and_loss_breakdown_series` | 产品级 P&L 拆解的时间趋势。 |

> **规律:** P&L 工具沿三个维度变化——**范围**(账户 / 产品)×**形态**(汇总 / 拆解)×**时间**(时点 / `_series` 趋势)。按问题选对应组合即可。

### 库存 (Inventory)

| 工具 | 作用 |
|---|---|
| `get_inventory_values` | 各产品当前的亚马逊库存数量,含履约上下文。 |
| `get_fba_inventory_health_status` | 各产品 FBA 库存健康度——可供天数与库存风险指标。 |
| `get_sales_velocity` | 一段时间内的销量(销售速度),按天 / 周期分桶。 |

### 广告查询 (Ads Query)

一个自描述、schema 运行时发现的查询流程——是上面调研工具的分析型对应物。它以固定的 5 步顺序执行:

| 步骤 | 工具 | 作用 |
|---|---|---|
| 1 | `get_ads_query_platforms` | 列出你可用的广告平台。 |
| 2 | `get_ads_query_list` | 获取某平台的广告数据层级。 |
| 3 | `get_ads_query_schema` | 自描述 schema——维度、指标、过滤器、粒度。 |
| 4 | `get_ads_query_materials` | 将名称(如广告活动)解析为查询所需的 ID。 |
| 5 | `execute_ads_query` | 执行查询并将列 + 行内联返回(终止步骤)。 |

**标准流程:** `get_ads_query_platforms` → `get_ads_query_list` → `get_ads_query_schema` → `get_ads_query_materials` → `execute_ads_query`。

---

## 示例提示词

**关键词调研:**

> 用 Helium 10,给我看 ASIN B0XXXXXXXX 排名的高频关键词,按搜索量排序。

> 用 Helium 10,把种子词 "yoga mat" 扩展成相关关键词,带上搜索量和竞争度。

**产品调研:**

> 用 Helium 10,在 "kitchen" 类目里找月营收超过 $10k、评论少于 200 的在售产品。

**利润:**

> 用 Helium 10,给我上个月的账户 P&L 汇总,然后按产品拆解。

> 用 Helium 10,画出 ASIN B0XXXXXXXX 最近 90 天的产品级利润趋势。

**库存:**

> 用 Helium 10,我哪些产品有断货风险?给出可供天数和近期销售速度。

**广告查询:**

> 用 Helium 10 Ads Query,按天给我看主账户最近 7 天的广告花费、ACoS 和 ROAS。

Agent 会在运行时发现 schema,在关键参数处请你确认,并将结果直接返回到对话中。

---

## 故障排查

**浏览器打开了 Helium 10 MCP 的 gateway 页面。**
该 gateway URL 是 MCP 端点,不是网页。如果它在浏览器里打开了,说明客户端尝试认证但没有完成 OAuth 流程,于是回退到了 URL。确认客户端配置中只有 URL、**没有 `headers` 块**,这样它会显示一个真正的 **Connect** 按钮;然后彻底重启客户端(对某些客户端而言,刷新窗口不够)。

**工具没有出现。**
几乎都是连接或认证问题:彻底重启 MCP 客户端,确认 OAuth 会话有效。工具列表还受账户权限限制——你只会看到自己有权限的 toolset 和平台。

**连接断了,要求我重新登录。**
长时间不活动后属正常现象——OAuth refresh token 会过期。重新走一遍浏览器流程即可恢复。

**我需要 headless / CI / API Key 认证。**
暂不支持。Helium 10 MCP 目前仅支持 OAuth;基于 Token 的认证已在规划中。当前请使用能完成浏览器 OAuth 流程的客户端。

**Agent 在一个任务里混用了多个 toolset。**
Agent 每个任务应只用一个 toolset。如果你看到调研类调用出现在 Ads Query 流程中间(或反过来),请在新的一轮对话里重新说明意图。

---

## 安全与限制

- OAuth 的 access token + refresh token 在服务端仅以哈希形式存储。
- 每个凭证与一个 Helium 10 用户 1:1 绑定。数据访问范围受限于该用户的账户权限。
- 凭证的签发、使用与撤销均写入审计日志。
- 撤销立即生效——下一个携带已撤销凭证的请求会被拒绝。
- 传输仅走 TLS。

##其他

API docs: https://mcp.helium10.com/explorer/index.html

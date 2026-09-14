# Quant Agent Skills · 量化智能体技能集

> 让 Agent **记住**你的每一次判断 —— **17 个分析技能 + 5 个记忆核心技能**
> 数据是商品，计算是工具，**记忆才是资产**。

[English](#english) | 中文

---

## 为什么记忆才是核心

大多数量化工具集只做两件事：**取数**（行情/资金/财报）和**计算**（因子/回测/因果）。这两层都是商品：

| 层 | 可替代性 | 谁能做 |
|---|---|---|
| 取数 | **很高** | Tushare / akshare / 东财 / 同花顺 |
| 计算 | **中** | qlib / statsmodels / econml 全开源 |
| **记忆** | **很低** | 记录"你的决策 → 你的结果"，无人能复制 |

**同样的工具，有没有记忆是两种产品：**

```
❌ 无记忆 Agent：
   你：现在市场什么状态？
   Agent：regime = range，置信度 0.7          ← 一个孤立的数字
   你：（下个月再问，它还是只会说这一句）

✅ 有记忆 Agent：
   你：现在市场什么状态？
   Agent：（先检索）你在类似情境下判断过 4 次，2 次正确，且都在「切换期」看错
         → regime = range（置信度已下调）
         → 本次判断已存入记忆，5 个交易日后可回来对账
```

**用一年后**：前者给你 365 个孤立读数；后者给你一套**只属于你的交易经验库**——这才是别人拿不走的。

## 22 个技能

### 🧠 记忆核心（5）—— 产品灵魂

| # | 技能 | 用户会这么说 | 作用 |
|---|---|---|---|
| 1 | [`trade-journal`](skills/trade-journal/) | 记一下我这次买的 / 把我的判断存下来 | 把交易与判断结构化落账（决策/理由/预期/验证期） |
| 2 | [`review-loop`](skills/review-loop/) | 回顾一下我这段时间的操作 / 我上次判断对了吗 | 找到待验证决策 → 对照实际 → **回填结果** → 提炼教训 |
| 3 | [`similar-cases`](skills/similar-cases/) | 我以前遇到过类似情况吗 | 检索历史相似情境的判断与结果，对照当前 |
| 4 | [`what-if`](skills/what-if/) | 如果当时没卖会怎样 / 这两个方案哪个好 | 反事实对比（决策前比较选项，决策后复盘未选路径） |
| 5 | [`memory-health`](skills/memory-health/) | 我的记忆库怎么样 / 我的预测准吗 | 记忆库体检：预测命中率、矛盾解决、错误经验软失效 |

### 🔬 分析技能（17）—— 每个都带**记忆回路**

每个分析技能都是「**前置检索 → 执行计算 → 后置写记忆 → 输出带历史参照**」的闭环。

| 类别 | 技能 | 用户会这么说 |
|---|---|---|
| 🛡 持仓风险 | [`portfolio-checkup`](skills/portfolio-checkup/) [`rebalance-plan`](skills/rebalance-plan/) [`volatility-alert`](skills/volatility-alert/) | 我的持仓有风险吗 / 该怎么调仓 / 波动会不会变大 |
| 📈 市场状态 | [`market-regime`](skills/market-regime/) [`trend-turning`](skills/trend-turning/) | 现在什么状态 / 是趋势还是回调 |
| 🔮 走势预测 | [`kline-forecast`](skills/kline-forecast/) [`next-day-prediction`](skills/next-day-prediction/) | 未来几天怎么走 / 明天大概怎么走 |
| 📡 消息发现 | [`news-radar`](skills/news-radar/) | **今天有什么消息** / 最近发生了什么 / 扫一下消息面 |
| ⚡ 事件因果 | [`event-impact`](skills/event-impact/) [`event-counterfactual`](skills/event-counterfactual/) [`event-propagation`](skills/event-propagation/) [`sector-linkage`](skills/sector-linkage/) | 这消息有影响吗 / 这事拉动多少 / **会传导到谁** / 谁带谁 |
| 🔍 结论检验 | [`news-credibility`](skills/news-credibility/) | 这个结论靠谱吗 / 这信号是真的吗 / 是不是过拟合 |
| 🧪 因子策略 | [`factor-vitality`](skills/factor-vitality/) [`factor-production-check`](skills/factor-production-check/) | 因子还能用吗 / 能上实盘吗 |
| 🔄 批量运维 | [`batch-screening`](skills/batch-screening/) [`task-status`](skills/task-status/) | 帮我扫一批 / 任务跑完了吗 |

## 快速开始

### 1. 接入 MCP 工具（**至少挂记忆 + 数据 + 计算三组**）

> ⚠️ 其中 `news-radar` 与 `event-propagation` 依赖两个**自建服务**（[news-mcp](https://github.com/JingxuanC/news-mcp)、
> [causal-chain-mcp](https://github.com/JingxuanC/causal-chain-mcp)），不在 hub 上，需要自己 clone + `make serve`。
> 其余 20 个技能只需 hub 的 `data` / `alpha` / `memory` 三组。详见 [`mcp-config/README.md`](mcp-config/README.md)。

```jsonc
{
  "mcpServers": {
    "mcphub-memory": {   // 🧠 记忆层（核心，必挂）
      "type": "streamable-http",
      "url": "https://causal-memory.com/hub/mcp/memory",
      "headers": { "Authorization": "Bearer YOUR_MCPHUB_KEY" }
    },
    "mcphub-data": {     // 📊 数据层（技能自动取数，必挂）
      "type": "streamable-http",
      "url": "https://causal-memory.com/hub/mcp/data",
      "headers": { "Authorization": "Bearer YOUR_MCPHUB_KEY" }
    },
    "mcphub-alpha": {    // 🔬 计算层（因子/因果/预测）
      "type": "streamable-http",
      "url": "https://causal-memory.com/hub/mcp/alpha",
      "headers": { "Authorization": "Bearer YOUR_MCPHUB_KEY" }
    }
  }
}
```

> 只挂 `alpha`（计算）会退化成"一次性计算器"——**记忆才是这个仓库的价值**。
> 各客户端详细配置见 [`mcp-config/`](mcp-config/)（Claude Code / Cursor / Cline / DSH）

### 2. 挂载技能

```bash
cp -r skills/* ~/.dsh/skills/          # DSH
cp -r skills/* ~/.claude/skills/       # Claude Code
```

### 3. 用起来

```
你：记一下，我今天在 002371 上减了半仓，理由是看到变点信号，预期 5 日内回调
Agent：（trade-journal）已记录：决策=减仓 002371 至 50%｜理由=变点信号｜预期=5日内回调｜验证期=5个交易日

—— 5 天后 ——

你：回顾一下我上周的操作
Agent：（review-loop）1 条待验证决策：002371 减仓
       实际：5 日内 -3.2%（低于预期幅度）→ 回填结果
       教训：变点信号确实提示了回调，但幅度判断偏乐观
       已写入记忆。你在「变点信号」类情境下历史 3 次，2 次方向对、幅度均偏乐观
```

## 依赖的工具（111 个 · 4 个组）

| 组 | 端点 | 工具数 | 作用 |
|---|---|---|---|
| **`memory`** | `/hub/mcp/memory` | 17 | 🧠 **记忆层**：决策记账、因果检索、反事实、预测校准 |
| **`data`** | `/hub/mcp/data` | 59 | 📊 数据层：A 股全维（行情/资金/龙虎榜/涨停池/公告/研报/期权/两融）+ 美股宏观 |
| **`alpha`** | `/hub/mcp/alpha` | 31 | 🔬 计算层：因子挖掘（17）+ 因果推断（9）+ K 线预测（5） |
| `match` | `/hub/mcp/match` | 4 | fetch / 时间 / 顺序思考（可选） |

<details>
<summary>展开：memory 组 17 个记忆工具（核心）</summary>

**写入**：`record_decision`（决策→结果因果边）`record_fact`（稳定事实）`remember`（自动提取）

**检索**：`search_causal` `search_facts` `search_memory`（融合）`search_patterns`（跨任务模式）`causal_directory`（最近决策）

**推理**：`counterfactual_query`（选项对比）`intervention_query`（行动前预测）`trace_cause` / `trace_cause_chain`（根因反推）`reconstruct_lesson`（教训重建）

**维护**：`prediction_report`（预测校准）`resolve_updates`（矛盾解决）`invalidate_decision` / `invalidate_pattern`（错误经验软失效）

</details>

<details>
<summary>展开：data 组 59 个 · alpha 组 31 个</summary>

**data · A 股（45）**：`get_a_realtime` `get_a_hist` `get_a_intraday` `get_a_indices` `get_a_fund_flow_120d` `get_a_fund_flow_minute` `get_a_north_flow` `get_a_margin` `get_a_dragon_tiger` `get_a_limit_up_pool` `get_a_broken_board` `get_a_board_emotion` `get_a_zt_reason` `get_a_hot_rank` `get_a_news` `get_a_announcements` `get_a_reports` `get_a_eps_forecast` `get_a_financials` `get_a_holder_num` `get_a_lockup_expiry` `get_a_option_greeks` …（完整清单见 `mcp-config/`）

**data · 全球（14）**：美股行情/基本面/内部交易、Reddit、StockTwits、FRED 宏观、Polymarket 预测市场等

**alpha · 因子（17）**：`compute_factors` `factor_backtest` `factor_oos_check` `factor_recent_ic` `factor_tearsheet` `ml_train_rolling` `ml_predict` `ml_metrics` `portfolio_optimize` `regime_detect` `change_point` `vol_forecast` `job_status` …
**alpha · 因果（9）**：`event_study` `event_study_batch` `causal_impact` `refute` `learn_graph` `granger_te` `pcmci_discover` `dml_cate`
**alpha · 预测（5）**：`kronos-forecast_kline` `kronos-forecast_signal` `kronos-forecast_batch` `kronos-model_info`

</details>

## 技能格式

标准 Agent Skills 格式（Markdown + YAML frontmatter），兼容 DSH / Claude Code / Cursor：

```markdown
---
name: market-regime
description: 当用户问当前市场状态时使用。用户可能说「现在什么状态」「该进攻还是防守」。
---

## 何时使用
## 记忆回路（前查后记）   ← 本仓库的核心差异
## 工具链
## 输出格式（含「历史参照」必填小节）
## 合规边界
```

## ⚠️ 合规声明

**本仓库只提供量化计算的工具编排与个人决策记录，不提供证券投资建议。**

- ✅ 允许：风险指标、统计显著性、最优权重、**你自己的历史决策记录与命中率**
- ❌ 不输出：买入/卖出建议、目标价、收益承诺
- 记忆记录的是**用户自己的决策与结果**，不是对他人的建议
- 详见 [DISCLAIMER.md](DISCLAIMER.md)

## License

MIT

---

<a id="english"></a>
## English

**Quant Agent Skills** — 20 agent skills (15 analysis + 5 memory-core) that turn 111 quantitative MCP tools into plain-language workflows.

**The core insight: data and computation are commodities, memory is the asset.** Every analysis skill is a closed loop: *retrieve past episodes → compute → record this decision → report with historical reference*. After a year you don't have 365 isolated readings — you have a personal trading experience base nobody can copy.

**Compliance**: tool orchestration and personal decision journaling only — no investment advice. See [DISCLAIMER.md](DISCLAIMER.md).

MIT License.

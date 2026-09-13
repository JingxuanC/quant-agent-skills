# Quant Agent Skills · 量化智能体技能集

> 让不会写提示词的交易员，也能用上 Agent 级的量化研究能力。
> **15 个开箱即用的技能**，把 35 个量化 MCP 工具封装成"一句话就能用"的研究流程。

[English](#english) | 中文

---

## 为什么做这个

大多数量化 MCP 工具集的现状：

```
工具方提供：35 个原子工具 + API key
用户需要  ：一个答案
中间的鸿沟：会配 MCP 客户端 + 会写提示词 + 会组合工具链
结果      ：只有少数会 Agent 的用户跨得过去
```

**这个仓库就是那座桥。** 每个技能 = 一段可复用的指令，告诉 Agent：
- 用户说哪些话时该用它（不用背提示词）
- 依次调用哪些工具、传什么参数
- 输出成什么结构
- 哪些结论**不能**给（合规边界）

## 15 个技能

| # | 技能 | 用户会这么问 | 背后工具链 |
|---|---|---|---|
| 1 | [`portfolio-checkup`](skills/portfolio-checkup/) | 我的持仓有风险吗 / 仓位重不重 | `portfolio_optimize` + `vol_forecast` + `regime_detect` |
| 2 | [`rebalance-plan`](skills/rebalance-plan/) | 该怎么调整仓位 | `portfolio_optimize`(hrp) + `regime_detect` |
| 3 | [`volatility-alert`](skills/volatility-alert/) | 我的票波动会不会变大 | `vol_forecast` + `change_point` |
| 4 | [`market-regime`](skills/market-regime/) | 现在市场什么状态 | `regime_detect` + `change_point` |
| 5 | [`trend-turning`](skills/trend-turning/) | 这波是趋势还是回调 | `change_point` + `regime_detect` |
| 6 | [`kline-forecast`](skills/kline-forecast/) | 未来几天怎么走 | `kronos-forecast_signal` + `model_info` |
| 7 | [`event-impact`](skills/event-impact/) | 这个消息对股价有影响吗 | `event_study` + `refute` |
| 8 | [`event-counterfactual`](skills/event-counterfactual/) | 这事到底把股价拉动了多少 | `causal_impact` + `event_study` |
| 9 | [`news-credibility`](skills/news-credibility/) | 这个结论靠谱吗 | `refute`（三重反驳） |
| 10 | [`sector-linkage`](skills/sector-linkage/) | 这两个板块谁带谁 | `granger_te` + `learn_graph` + `pcmci_discover` |
| 11 | [`factor-vitality`](skills/factor-vitality/) | 我这个因子还能用吗 | `factor_recent_ic` + `factor_tearsheet` |
| 12 | [`factor-production-check`](skills/factor-production-check/) | 这因子能上实盘吗 | `factor_oos_check` + `factor_backtest` |
| 13 | [`next-day-prediction`](skills/next-day-prediction/) | 明天大概怎么走 | `ml_predict` + `ml_metrics` |
| 14 | [`batch-screening`](skills/batch-screening/) | 帮我扫一批股票 | `event_study_batch` / `kronos-forecast_batch` / `learn_graph` |
| 15 | [`task-status`](skills/task-status/) | 我提交的任务跑完了吗 | 各 `job_status` |

每个技能都包含：**触发话术 → 轻问/深问分支（控制成本）→ 工具调用顺序 → 输出规范 → 合规边界**。

## 快速开始

### 1. 接入量化 MCP 工具

技能依赖 [mcphub](https://causal-memory.com) 提供的 35 个工具（因子挖掘 / 因果推断 / K 线预测 / 组合优化）。

**Claude Code / Cursor / Cline 等任意 MCP 客户端**：

```jsonc
{
  "mcpServers": {
    "mcphub": {
      "type": "streamable-http",
      "url": "https://causal-memory.com/hub/mcp/alpha",
      "headers": { "Authorization": "Bearer YOUR_MCPHUB_KEY" }
    }
  }
}
```

**DSH（DeepSeek Harness）**：见 [`mcp-config/`](mcp-config/)

### 2. 挂载技能

把 `skills/` 下的目录放进你 Agent 的技能目录：

```bash
# DSH
cp -r skills/* ~/.dsh/skills/

# Claude Code
cp -r skills/* ~/.claude/skills/
```

### 3. 直接说人话

```
你：我的持仓有风险吗？
Agent：（自动命中 portfolio-checkup 技能）
     集中度 62%（前三大）
     波动率 5 日上行 18%
     当前 regime: range → 仓位建议区间 40-60%
     ⚠️ 以上为量化计算结果，不构成投资建议
```

## 依赖的工具（35 个）

<details>
<summary>展开完整清单</summary>

**因子挖掘（17）**
`compute_factors` `predict` `factor_execute` `factor_backtest` `factor_oos_check` `factor_daily_compute` `factor_recent_ic` `factor_tearsheet` `ml_train_rolling` `ml_predict` `ml_metrics` `portfolio_optimize` `regime_detect` `change_point` `vol_forecast` `update_data` `job_status`

**因果推断（9）**
`event_study` `event_study_batch` `causal_impact` `refute` `learn_graph` `granger_te` `pcmci_discover` `dml_cate` `job_status`

**K 线预测（5）**
`kronos-forecast_kline` `kronos-forecast_signal` `kronos-forecast_batch` `kronos-model_info` `kronos-job_status`

**其他（4）**
`factor-miner-compute_factors` `factor-miner-predict` `causal-*` 命名空间工具

</details>

## 技能格式

标准 Agent Skills 格式（Markdown + YAML frontmatter），兼容 DSH / Claude Code / Cursor：

```markdown
---
name: portfolio-checkup
description: 当用户想评估持仓风险（集中度/波动率/回撤）时使用。
---

## 何时使用
## 工具链
## 轻问 vs 深问
## 输出格式
## 合规边界
```

## ⚠️ 合规声明

**本仓库只提供量化计算的工具编排，不提供证券投资建议。**

- ✅ 允许输出：风险指标、统计显著性、最优权重、波动率预测等**计算结果**
- ❌ 不输出：买入/卖出建议、目标价、收益承诺
- 使用者应自行判断，并遵守所在地区法律法规
- 详见 [DISCLAIMER.md](DISCLAIMER.md)

## License

MIT

---

<a id="english"></a>
## English

**Quant Agent Skills** — 15 ready-to-use agent skills that wrap 35 quantitative MCP tools (factor mining, causal inference, K-line forecasting, portfolio optimization) into workflows any trader can invoke in plain language.

Each skill ships with: trigger phrases, light/deep query branches (cost control), tool call order, output spec, and compliance boundaries.

**Compliance**: this repo provides tool orchestration only — no investment advice. See [DISCLAIMER.md](DISCLAIMER.md).

MIT License.

---
name: event-counterfactual
description: 用反事实方法估算「某个事件到底给股价贡献了多少」——如果没有这个事件，股价本会怎么走。当用户说「这事到底把股价拉动了多少」「没有这个利好会怎样」「净影响是多少」「这条消息算不算已经兑现了」时使用。
---

# 事件反事实评估

## 何时使用

命中以下任意一种说法就调用本技能：

- 「这个利好到底把股价拉动了多少？」
- 「如果没有这个事件，会怎样 / 会跌多少？」
- 「净影响是多少？剔除大盘之后还剩多少？」
- 「这消息是不是早就 price in 了，还有多少没兑现？」
- 「涨的这部分里，多少是事件贡献的，多少是大盘带的？」
- 「帮我算一下这件事对 XX 的累计影响，要显著性」


## 轻问 vs 深问（成本控制）

**轻问（1 个工具，优先默认走这条）**
- `mcp__causal__causal_impact`(symbol, event_date, klines, benchmark) —— 一次拿到 `cumulative_impact` / `avg_daily_impact` / `p_value` / `ci_95` / `method`
- 适用：用户只想快速知道「这事件净拉动大概几个点、显不显著」。回复 5-8 行即可。

**深问（完整链路 + 结构化报告）**
1. `mcp__causal__causal_impact` 主估计（BSTS/OLS 对冲反事实，给累计影响与区间）
2. `mcp__causal__event_study` 交叉验证（市场模型 CAR + t 检验，给 car_1d/3d/5d 与 event 前漂移）
3. 可选 `mcp__causal__refute`（传事件研究的 car_1d/3d/5d + p_value 做安慰剂 / 随机共因 / 子样本三个反驳，拿 A-F 评级）
4. 多事件批量：`mcp__causal__event_study_batch`（一次提交多个事件）
5. 前置核对：让用户确认事件日期与事件源（从新闻/公告原文核对，日期错一天结论就废）

**哪一步最贵**：
- `mcp__causal__event_study_batch` 是**最重**的一步 —— 多个事件在一个子进程里跑（默认 `timeout=120` 秒），必须先提交、再轮询，**不要同步傻等**。
- 单事件 `mcp__causal__event_study` 中等成本；`mcp__causal__causal_impact` 轻量，是默认主力。
- 两个估计方法都要**至少 60 个交易日的估计窗口**，klines 给不够会直接降级或返回零值占位（`status != 'ok'`），这不是工具坏了，是数据不够。

## 工具链

### 第 0 步：先确认「事件三要素」
调工具前必须和用户对齐这三样，缺一项就先问：
1. **标的**（如 `sz002916` / `sh600183`）
2. **事件日期** `event_date`（`YYYY-MM-DD`，非交易日会自动取最近交易日）
3. **事件是什么**（一句话：业绩超预期 / 大额定点 / 政策落地 / 大客户砍单）—— 影响解读口径，也方便和新闻/公告里的事件对上号

### 第 1 步：准备 klines 与 benchmark（必须由调用方提供）
本技能**不会自己去取数据**。klines 与 benchmark 都由调用方传入，数据来源二选一并明确告知用户：

- **用户提供**：让用户贴标的日线 + 对应指数日线（周期必须一致、日期必须对齐）。
- **系统数据源**：从行情/盯盘/因子侧已有的日 K 读入后再传入，并在输出里注明数据来源与截止日期。

格式（按日期升序）：
```json
klines    = [{"date":"2026-01-02","close":0}, ...]
benchmark = [{"date":"2026-01-02","close":0}, ...]   // 沪深300 / 中证1000 等指数
```

**窗口要求（关键，给不够就算不准）：**
- `mcp__causal__causal_impact`：事件前需要一段基准窗口做对冲回归外推；建议**事件前 ≥ 60 个交易日 + 事件后 ≥ 5 个交易日**。
- `mcp__causal__event_study`：估计窗固定在 (-60, -6)，再算 CAR (0,1)/(0,3)/(0,5)，所以 klines 至少要**覆盖事件前 60 日到事件后 5 日**。
- `benchmark` 与 `klines` 要**等长对齐**；`event_study` 允许缺失但按零收益处理，`causal_impact` 允许缺失但会影响基准质量 —— 缺失就如实说明，别硬凑。

### 第 2 步：主估计 —— 反事实影响（默认主力）
```
mcp__causal__causal_impact(
  symbol     = "sz002916",
  event_date = "2026-03-12",
  klines     = <上面的 klines 数组>,
  benchmark  = <上面的 benchmark 数组>
)
```
返回关键字段：
- `cumulative_impact` —— 事件后累计影响（超额收益口径）
- `avg_daily_impact` —— 事件后日均影响
- `p_value` / `significant` —— 经验 p 值（事件前残差同长度滚动窗口的双侧置换分位）与显著性
- `ci_95` —— 95% 区间
- `method` —— `bsts`（已装 pycausalimpact，走真 BSTS）或 `ols_hedge`（降级为 OLS 对冲回归），必须原样报给用户
- `alpha` / `beta` / `r_squared` —— 对冲回归参数与拟合优度（`beta` 是相对基准的暴露）

**怎么读**：`cumulative_impact` 是剔除基准（大盘/行业）之后**剩下的那部分**，也就是"事件贡献"。`beta` 明显偏离 1 时，要对用户说明该标的本身波动就比指数大。

### 第 3 步（仅深问）：交叉验证 —— 事件研究 CAR
```
mcp__causal__event_study(
  symbol           = "sz002916",
  event_date       = "2026-03-12",
  klines           = <同上>,
  benchmark        = <同上>,
  sector_benchmark = <可选：板块指数日线，给了就走"市场+板块"多因子模型>
)
```
返回：`car_1d` / `car_3d` / `car_5d` / `car_pre1d`（事件前一日漂移，用来看是否提前走漏）、`alpha` / `beta_market`(可选 `beta_sector`) / `r_squared` / `sigma` / `t_stat` / `p_value` / `is_significant` / `direction` / `impact_score` / `confidence` / `status`。

**`status != 'ok'` 时其余字段是零值占位** —— 不要把这些 0 当成"影响为零"报出去，要说明原因是窗口不足/数据缺失。

**与第 2 步互验**：两个方法的符号与量级应大致同向。若一个显著一个不显著、或符号相反，**必须把这个分歧如实写进结论**，不要挑一个好看的说 —— 这通常说明事件影响不稳健。

### 第 4 步（可选）：反事实验证
```
mcp__causal__refute(
  chain = {
    symbol: "sz002916", event_id: <可选>, event_title: <可选>,
    car_1d: <第3步>, car_3d: <第3步>, car_5d: <第3步>,
    p_value: <第3步>, direction: <第3步>
  }
)
```
三个反驳逐项给 `robust` / `refuted` + 总体评级 `grade`（A-F）与 `recommendation`。评级低（D-F）时，前面算出的影响**只能当参考**，要明确降级表述。

### 第 5 步（仅多事件批量）：先提交、再轮询
```
mcp__causal__event_study_batch(
  events = [
    {"event_id": 1, "symbol": "sz002916", "event_date": "2026-03-12", "klines": [...], "benchmark": [...]},
    {"event_id": 2, "symbol": "sh600183", "event_date": "2026-03-15", "klines": [...], "benchmark": [...]},
    ...
  ],
  timeout = 120
)
```
**批量是重任务：先提交拿 `job_id`，再用 `mcp__causal__job_status(job_id = "<job_id>")` 轮询 `queued / running / done / error`，不要傻等。** 期间可以先给用户报进度或先讲单事件的主估计结论；`done` 后读 `result`。每个事件返回字段同 `event_study`，并带 `event_id` / `symbol` / `event_date` 回显；失败的事件如实说明，**不补造数值**。

## 记忆回路（前查后记）

| 时机 | 动作 | 工具 |
|---|---|---|
| 执行前 | 查同类事件（同一事件类型：业绩超预期 / 大额定点 / 政策落地 / 大客户砍单，以及同一标的或同板块）历史上算出的反事实影响与**后续验证结果**：当时 `cumulative_impact` 是否显著、`method` 是 `bsts` 还是降级的 `ols_hedge`、以及事后是否真的兑现到这个量级 | `search_causal`（query=`"事件反事实 累计影响 {事件类型} {symbol}"`，task_tag=`event-counterfactual`） |
| 执行后 | 写本次反事实估计的核心结论：`cumulative_impact` / `avg_daily_impact`、`p_value` 与是否显著、`method`、`beta`、CAR 交叉验证是否同向、`mcp__causal__refute` 评级（若跑了） | `record_decision`（decision=`"反事实估计：{symbol} 于 {event_date} 的 {事件类型}，cumulative_impact={x}%，p={p}，method={method}，显著={是/否}"`，outcome="待验证"，context=`"事件类型={}，窗口=事件前{}日+事件后{}日，benchmark={}，事件是否提前 price in（car_pre1d={}）"`） |
| 复检时 | 事件后 **T+5 / T+10 交易日**结算窗口走完、或用户回来问「上次那事件后来兑现了吗」时，用**同一条 decision 文本**再记一次真实结果，形成可对照的闭环 | `record_decision`（同一条 decision，outcome=`"实际累计超额收益={实测}%，与估计{x}% 偏差={}pp；当时判定显著/不显著，事后{兑现/未兑现}"`） |

本技能的历史参照维度是**「同类事件的反事实估计 → 后续验证」这一对**：同一种事件类型在类似标的上，估计出的净影响有多少能在 T+5/T+10 真的兑现，以及不显著的事件事后是否果然没有可辨识的贡献。这个记忆对用户的价值在于把一次性算数变成可校准的经验——若历史上「显著」的反事实估计多数兑现，「不显著」的多数确实无贡献，之后再给出显著性判定时就有了先验命中率；反之则提示某些事件类型（例如易被提前 price in 的利好）的反事实估计系统性偏乐观，应当主动降级表述。跨事件批量（第 5 步）跑完后，逐条按上面的方式落库，可积累出按事件类型的置信度分布。

## 输出格式

Agent 回复必须包含以下小节，字段名可中文化但不得缺项：

1. **事件与标的**：`symbol`、`event_date`、事件一句话描述、事件来源（用户口述 / Silver 新闻 / causal_chains）。
2. **数据与窗口**：klines 区间、根数、benchmark 用的是哪个指数、数据来源、估计窗与事件窗。
3. **反事实估计（主）**：`cumulative_impact`、`avg_daily_impact`、`ci_95`、`p_value` + 是否显著、`method`（`bsts` / `ols_hedge` 原样标注）、`beta` / `r_squared`。
4. **交叉验证（深问）**：`car_1d` / `car_3d` / `car_5d` / `car_pre1d`、`t_stat` / `p_value` / `is_significant`、`status`；两个方法的**一致或分歧说明**。
5. **稳健性（若跑了）**：`mcp__causal__refute` 的三项判定 + 总体 `grade`、`recommendation`。
6. **口径声明（固定收尾，不可省略）**：
   > 以上为**统计因果估计**（反事实口径：剔除基准后属于该事件的那部分超额收益），**非承诺**，因果推断不等同于确定性因果，不构成任何投资建议。
7. **显著性与方法闸门（强制）**：
   - 不显著（`p_value` 偏大 / `significant=false`）或 `ci_95` 跨 0 → **必须明写「事件影响不显著，无法确认净贡献」**，不得用方向性话术暗示事件有效。
   - `method` 是 `ols_hedge`（降级）时必须点明"未走 BSTS，区间估计精度有限"。
   - 交叉验证分歧 → 必须写明"两方法结论不一致，影响不稳健"。
8. **历史参照**（必填小节）：若检索到同类历史，写明「类似情境 N 次，其中 X 次 <本技能的判断维度> 正确」；
   无历史则写「首次记录此类情境，本次判断已存入记忆供将来对照」。

## 合规边界

- 只输出**统计计算结果**：累计影响、日均影响、区间、p 值、方法、CAR、稳定性评级。
- **不输出**：买入/卖出/加仓/减仓建议、目标价、止盈止损价位、仓位比例、收益承诺、胜率承诺、确定性措辞（"必涨""稳了""一定"）。
- 本方法给出的是**统计因果估计，不是确定的因果事实**：必须标"反事实口径 + 非承诺"，不得把 `cumulative_impact` 说成"事件一定值这么多钱"。
- 不把估计结果包装成"信号"或"推荐"；用户若追问该不该买，明确说明超出本技能边界，可转研究员/风控工具做逻辑与风险层面的分析。
- 数据不足（`status != 'ok'`、klines 窗口不够、benchmark 缺失）时**如实说"数据不足，估计不可靠"**，不得用零值占位或少量数据凑结论。

## 常见问题

1. **「为什么影响算出来不显著？」**
   很常见：事件被提前 price in（看 `car_pre1d` 是否已有明显漂移）、事件本身被大盘行情淹没（看 `beta` 与 `r_squared`）、或样本窗口太短。正确回答是**「影响不显著，无法确认净贡献」**，而不是挑一个显著的口径硬讲 —— 硬凑显著性是本技能最容易犯的错。

2. **「没有 benchmark 能不能算？」**
   `benchmark` 缺失时 `event_study` 按零收益处理、`causal_impact` 的基准质量会崩，算出来的只是**原始涨跌幅**，不是剔除大盘后的净影响。要么让用户补指数日线，要么在输出里明确标注"未对冲基准，结论仅为原始收益口径"。

3. **「批量跑多个事件卡住了？」**
   `mcp__causal__event_study_batch` 是重任务：拿到 `job_id` 后用 `mcp__causal__job_status` 轮询，`queued/running` 期间先做别的事或报进度，**不要阻塞式死等**；单个事件失败不拖垮整批，结果里失败项如实说明。

4. **「klines 要多少根？」**
   `causal_impact` 建议事件前 ≥ 60 日 + 事件后 ≥ 5 日；`event_study` 估计窗固定 (-60, -6)，所以至少要覆盖事件前 60 日到事件后 5 日。给不够会降级或返回零值占位 —— 提前问清楚，比事后解释"数据不够"省事。

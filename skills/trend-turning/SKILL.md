---
name: trend-turning
description: 判断当前行情是趋势延续还是结构性转折。当用户说「这波是趋势还是回调」「见顶了吗」「见底了吗」「还能拿吗」「是不是要变盘了」时使用，输出变点日期、显著性与当前 regime。
---

# 趋势转折检测

## 何时使用
- 「这波是趋势还是回调？」
- 「见顶了吗？」
- 「还能拿吗？」
- 「是不是要变盘了？」
- 「跌了三天，是洗盘还是趋势坏了？」
- 「我这个票现在是什么状态，牛还是熊？」

## 轻问 vs 深问（成本控制）
- **轻问**（默认，几秒）：只跑 `factor-miner-change_point` 找最近 1 个变点，回一句「最近变点在 X 日，显著性 S，之后属于 Y 段」。只调 1 个工具。
- **深问**（结构化报告）：`factor-miner-change_point`（cusum_binseg 粗筛 + pelt 复核，取交集更可信）+ `factor-miner-regime_detect`（给 regime 结论与历史占比）。调 2 个工具，成本仍然是**低**。
- **最贵的一步**：`regime_detect` 的 `n_regimes=3` 会走 GaussianHMM 拟合（`hmmlearn` 已装时），比规则引擎慢一到两个数量级；不确定时先用 `n_regimes=2` 或直接接受内置规则引擎结果。**本技能不涉及回测/训练/批量任务**，因此通常不需要 job_status；只有在用户额外要求「顺带跑个因子回测或 ML 训练」时才走异步：先提交拿 `job_id`，再用 `factor-miner-job_status` 轮询（`job_status` 里传 `job_id`），不要傻等。

## 工具链
数据前提：**变点检测与 regime 都只吃 K 线价格序列，本技能不自动抓行情**。需要用户提供数据，或由上游行情工具/`factor-miner-ml_predict` 的 `klines_list` 传入。少于约 60 根 K 线时结论不可靠，少于 30 根直接拒答。

1. **找结构性变点** — `factor-miner-change_point`
   - 参数：`series`（必填，`[{"date": "2026-01-02", "value": 10.35}, ...]`，按时间**升序**，`value` 一般是收盘价或净值）、`method`（`"auto"` / `"cusum_binseg"` / `"pelt"`）、`max_bkps`（默认 5，日线建议 3-5；想要更细可加到 8，但噪声变点也会变多）。
   - 粗筛用 `method="cusum_binseg"`，复核用 `method="pelt"`；两种方法都指到同一位置的变点，关注度最高。
   - 返回：`change_points`（`[{date, index, significance}, ...]`）、`method`。`significance` 越高代表该点前后均值偏移越强。

2. **确认当前状态** — `factor-miner-regime_detect`
   - 参数：`klines`（必填，`[{"date": ..., "close": ..., "volume": ...}, ...]`，升序）、`n_regimes`（2 = 牛/熊，3 = 牛/熊/震荡，默认 3）。
   - 返回：`current_regime`、`regime_history`、`regime_stats`、`method`（`rule` 或 `hmm`）。**要如实告诉用户 `method` 是哪一个**：规则引擎或 HMM，结论口径不同。

3. **对齐时间轴** — 把步骤 1 的变点日期与步骤 2 的 `regime_history` 切换点对照：
   - 变点日期 ≈ regime 切换点 → 转折证据强。
   - 变点日期离 regime 切换点很远、或落在同一 regime 内部 → 只是段内波动，不构成转折。

4. **（可选）与用户观点互证** — `factor-miner-vol_forecast`（`klines`，`horizon`）看波动是否在放大，作为辅助旁证，不作为结论依据。

## 记忆回路（前查后记）

| 时机 | 动作 | 工具 |
|---|---|---|
| 执行前 | 查同类标的/相似情境下历史变点判断是否兑现：检索「变点判定为转折后，价格是否站稳新区间、regime 是否真切换」的旧记录 | `causal-memory-search_causal`（query=「趋势转折 变点判断 是否兑现 站稳新区间 regime 切换」，task_tag=`trend-turning`） |
| 执行后 | 写本次变点结论：decision = 「判定 <标的> 在 <最新数据日> 属于 趋势延续/疑似转折/已转折，变点日期 <D>、显著性 <S>、检测方法 <cusum_binseg/pelt>、regime=<标签>(method=<rule/hmm>)、变点与 regime 是否对齐」；outcome 先记「待验证」 | `causal-memory-record_decision`（decision=<本次结论>，outcome="待验证"，context=<情境特征>，relation=`caused`） |
| 复检时 | 变点之后又有 5-10 根新 K 线进来时（或用户回访「上次那个转折对不对」）回填真实结果：变点后价格是否站稳新区间、regime 是否真的切换、该变点是否被后续数据改写 | `causal-memory-record_decision`（同一条 decision，outcome=实际结果，relation 视情况取 `caused`/`no_effect`，context 补上新数据日） |

情境特征建议写全：标的、K 线根数、起止日期、`significance` 量级、`method`（`cusum_binseg`/`pelt`）、regime 的 `method`（`rule`/`hmm`）、以及是否与用户观点互证——这些正是让「本次判断」与「历史同类判断」可比、可对照的关键字段。

本技能的历史参照维度只有一个：**历次变点判断的对错**——即每次判定「已转折/疑似转折/趋势延续」之后，价格与 regime 是否真的按该方向走了。这个记忆对用户的价值在于：变点检测天然滞后且会被后续数据改写，单次结论看起来都很合理，只有累积「同类情境下判断对了多少次」才能让用户知道该给这次的变点日期和转折标签打多少折扣，而不是每次都当成新鲜结论。

## 输出格式
Agent 向用户交付时必须包含（缺一项即视为不合格）：

1. **结论一句话**：当前属于「趋势延续 / 疑似转折 / 已转折」，并注明判定依据是变点还是 regime。
2. **变点表**：`日期 | 位置(第N根K线) | 显著性 | 检测方法`，按时间倒序，最多 5 条。
3. **当前 regime**：`current_regime` + `method`（`rule`/`hmm`）+ 该 regime 在历史中的占比（来自 `regime_stats`）。
4. **一致性判断**：变点与 regime 切换是否对齐（对齐 / 部分对齐 / 不对齐）。
5. **数据口径**：使用的标的、K 线根数、起止日期。
6. **免责声明**：固定一句话——「以上为统计计算结果，变点检测是结构性回顾方法，不是对未来涨跌的预测。」

> **历史参照**（必填小节）：若检索到同类历史，写明「类似情境 N 次，其中 X 次 <本技能的判断维度> 正确」；
> 无历史则写「首次记录此类情境，本次判断已存入记忆供将来对照」。

## 合规边界
- **只输出统计计算结果**：变点日期、显著性、regime 名称、历史占比、方法名。
- **不输出买入/卖出/加仓/减仓建议**，不说「该走了」「可以抄底」。
- **不输出目标价、点位预测、收益率承诺**，不把 regime 标签翻译成「看涨/看跌多少」。
- 若涉及 Kronos 等预测模型或 ML 打分（`kronos-forecast_signal`、`factor-miner-ml_predict`），必须原样带上「**模型输出，非承诺**」，且只作为旁证，不能替代变点/regime 结论。
- 数据不足（< 30 根 K 线）时明确说「样本不足，无法给出可靠变点」，不要硬凑结论。

## 常见问题
1. **变点多不代表要变盘。** `max_bkps` 调大后每个小台阶都会被抓成一个点。判断标准是 `significance` 的量级和「变点后价格是否站稳新区间」，不是变点个数。只有 1 个高显著性点 + regime 同步切换，才叫转折。
2. **`regime_detect` 的 method 必须转述。** `hmm` 是拟合出来的隐状态，`rule` 是「20日动量 + 已实现波动」的阈值规则。用户拿这个结论去决策前，必须知道是哪种，否则容易把规则阈值当成模型结论。
3. **变点是滞后的、会被后续数据改写。** CUSUM/PELT 是事后分割，末尾几根 K 线的变点在数据更新后可能前移或消失。所以给日期时要说明「截至 <最新数据日> 的检测结果」，并提醒用户新数据进来后可重新跑一次。
4. **PELT 需要 `ruptures`，HMM 需要 `hmmlearn`。** 这两个库没装时工具会自动降级（PELT → 内置 CUSUM+二分分割，HMM → 规则引擎），结果仍然可用但口径变了——看 `method` 字段如实汇报即可。

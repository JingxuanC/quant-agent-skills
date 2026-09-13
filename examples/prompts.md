# 提问模板库 · Prompt Cookbook

> **不会写提示词？直接复制下面的句子。**
> 每一条都已对应到某个技能，说人话就能用。

---

## 🛡 持仓与风险

| 复制这句话 | 命中技能 | 你会得到 |
|---|---|---|
| 我的持仓有风险吗？ | `portfolio-checkup` | 集中度、波动率趋势、当前 regime 与仓位区间 |
| 我这仓位重不重？ | `portfolio-checkup` | 前三大权重占比 + 与最优配置的偏离 |
| 帮我看看我的组合合不合理 | `portfolio-checkup` | 同上，附风险分解 |
| 该怎么调整仓位？ | `rebalance-plan` | 当前权重 vs HRP 最优权重偏离表 |
| 我这些票里哪个风险最大？ | `portfolio-checkup` | 按波动率/集中度贡献排序 |
| 我的票波动会不会变大？ | `volatility-alert` | 未来 N 日波动率预测 + 变点检测 |
| 最近波动好大，要避险吗？ | `volatility-alert` | 波动率是否发生结构性上升 |

## 📈 市场状态

| 复制这句话 | 命中技能 | 你会得到 |
|---|---|---|
| 现在市场什么状态？ | `market-regime` | bull/bear/range 分类 + 仓位区间 |
| 该进攻还是防守？ | `market-regime` | regime + 历史同状态下的表现 |
| 这波是趋势还是回调？ | `trend-turning` | 结构性变点日期 + 显著性 |
| 是不是见顶了？ | `trend-turning` | CUSUM/PELT 变点检测结果 |
| 大盘什么时候变盘的？ | `trend-turning` | 变点位置（精确到交易日） |

## 🔮 走势预测（模型输出，非承诺）

| 复制这句话 | 命中技能 | 你会得到 |
|---|---|---|
| 未来几天怎么走？ | `kline-forecast` | Kronos 模型方向 + 预期收益 + 一致性置信度 |
| 明天大概怎么走？ | `next-day-prediction` | ML 次日预测分 + 模型近期 IC |
| 这票短期怎么看？ | `kline-forecast` | 多采样预测路径统计 |
| 预测一下这 5 天 | `kline-forecast` | pred_len=5 的方向与波动 |

## ⚡ 事件与因果

| 复制这句话 | 命中技能 | 你会得到 |
|---|---|---|
| 这个消息对股价有影响吗？ | `event-impact` | CAR 多窗口 + t 检验 p 值 + 显著性 |
| 这利好是不是已经 price in 了？ | `event-impact` | 事件窗口内的累计异常收益 |
| 为什么今天跌了？ | `event-impact` | 用当日事件做事件研究验证 |
| 这事到底把股价拉动了多少？ | `event-counterfactual` | 反事实累计影响 + 95% 区间 |
| 没有这个利好会怎样？ | `event-counterfactual` | BSTS 反事实基准线 |
| 这个结论靠谱吗？ | `news-credibility` | 三重反驳（安慰剂/随机共因/子集）+ A-F 评级 |
| 这个信号是真的吗？ | `news-credibility` | 稳健性验证 |
| 这两个板块谁带谁？ | `sector-linkage` | Granger 因果方向 + 传递熵 + lead-lag |
| A 涨了 B 会跟吗？ | `sector-linkage` | 领先滞后关系（统计层面） |

## 🧪 因子与策略

| 复制这句话 | 命中技能 | 你会得到 |
|---|---|---|
| 我这个因子还能用吗？ | `factor-vitality` | 近期 IC/IR + 衰减判断 |
| 因子失效了吗？ | `factor-vitality` | 与历史 IC 的对比 |
| 最近 IC 怎么样？ | `factor-vitality` | 滚动 IC 均值与稳定性 |
| 这因子能上实盘吗？ | `factor-production-check` | 样本外验证 + 衰减 + 淘汰建议 |
| 过拟合了吗？ | `factor-production-check` | 挖掘窗口 vs 纯 OOS 对比 |
| 帮我回测这个因子 | `factor-production-check` | qlib 全量回测（异步，给 job_id） |

## 🔄 批量与运维

| 复制这句话 | 命中技能 | 你会得到 |
|---|---|---|
| 帮我扫一批股票 | `batch-screening` | 批量事件研究/预测，统一表格排序 |
| 这几只哪个好？ | `batch-screening` | 按显著性排序的对比表 |
| 任务跑完了吗？ | `task-status` | job 状态（queued/running/done/error） |
| 刚才提交的好了吗？ | `task-status` | 结果摘要 |

---

## 💡 提问技巧（可选读）

1. **带上标的和数据**：技能需要 K 线数据，数据越全结果越准（如"用最近 250 天日线"）
2. **想快就说"大致"**：轻问只跑 1-2 个工具，几秒出结果；说"算不算显著/稳不稳"会触发深问（更慢更准）
3. **要报告就说"详细"**：如"给我一份详细的持仓体检"
4. **重任务用异步**：回测/训练类会说"已提交，job_id=xxx"，之后问"任务跑完了吗"即可

## ⚠️ 边界

技能输出的是**统计计算结果**（CAR、p 值、IC、权重、预测分），**不是买卖建议**。
任何决策请自行判断，并遵守所在地区法律法规。

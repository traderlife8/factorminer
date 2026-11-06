# 证据协议

FactorMiner 的证据报告将**预测、统计、风险与实现**诊断组合起来,**不**把单一分数当作 alpha 的证明。协议如下:

```text
点-in-time 数据与冻结的目标/宇宙
  → 显式 Pearson IC 与 Spearman RankIC
  → 序列依赖感知的不确定性与多重检验控制
  → purged/walk-forward 选择证据
  → 风格/行业中性化残差证据
  → 精确的组合换手、成本、冲击与容量
  → 独立的事件驱动 replay
  → 论文/实盘前向证据
```

FactorMiner 提供**时间数据集划分**、**滚动研究评估**、**CPCV/PBO**、**块自助检验**、**Benjamini-Hochberg FDR**、**Deflated Sharpe 实现**、**成本与容量模型**与**冻结基准 manifest**。`industry_evidence_v1` 把这些原语组合起来,加入显式 Pearson IC 与风格/行业信号中性化,同时保持历史的 `paper_ic_v2` 准入契约兼容。

## 信任层级

| 层 | 它回答什么 | 最低可信证据 | FactorMiner 状态 |
| --- | --- | --- | --- |
| 数据完整性 | 决策时信息是否可得? | 点-in-time 字段、退市、公司行动、源哈希、目标延迟 | 调用方契约与 manifest;vendor 真相仍然在外部 |
| 预测关联 | 信号是否排序/线性预测未来收益? | Pearson IC 与 Spearman RankIC(分时期、覆盖率、符号) | 两者都显式;遗留 `ic_*` 仍是 RankIC |
| 采样不确定性 | 在序列依赖下均值是否稳定? | IC 序列、Newey-West/HAC t 统计、块自助 CI | 已实现 |
| 选择卫生 | 赢家是否从多次试验中选出? | 完整试验计数、家族 p 值、FDR、DSR、CPCV 路径矩阵/PBO | 在提供完整输入时已实现 |
| 时间泛化 | 拟合是否在持仓评估的上游? | 训练/验证/测试冻结、purge label overlap、滚动制度 | 固定划分、滚动划分、CPCV 与 PBO 都存在 |
| 风险归因 | 信号是不是换皮风格/行业敞口? | 点-in-time 风格与行业敞口、原始与残差指标 | 开源的 OLS/WLS 残差化已实现;专有敞口估计**未**实现 |
| 经济实施 | 在 churn 与交易摩擦下能否存活? | 目标权重换手、成本曲线、价差/费用/冲击、容量 | 线性成本压力 + 更丰富的成本/容量模块 |
| 独立回放 | 另一执行引擎能否复现订单与 P&L? | 冻结权重/订单 + 日历、撮合、行为与约束 replay | **不**打包;推荐 LEAN/另一引擎做外部检查 |
| 前向真值 | 研究结束后行为是否延续? | 规则不变的论文/实盘 track record | 外部 |

报告会**显式**地把每个门槛标记为 `measured`、`partial`、`not_supplied` 或 `external_required`。**缺失的点-in-time 或风险模型输入不会被转换为令人放心的默认值**。

## 指标契约:名字不够

对资产 `i` 与决策期 `t`,令 `s[i,t]` 为因子值,`r[i,t+1]` 为已对齐的未来收益。

### IC 与 RankIC

```text
Pearson IC_t = PearsonCorr_i(s[i,t], r[i,t+1])
RankIC_t     = SpearmanCorr_i(s[i,t], r[i,t+1])
```

Pearson IC 检验截面线性关系,对信号幅值与离群点敏感。RankIC 检验**单调顺序**,对单调变换不变。因子挖掘系统通常优化秩,RankIC 可能看起来不错但信号幅值不稳定;反之亦然。**同时报告两者**。

FactorMiner 历史上称其 Spearman 序列为 `ic_series`。该契约在保存的产物与准入中**保持不变**。新产物加入 `ic_definition: spearman_rank`、`rank_ic_*` 与 `pearson_ic_*` 字段。

### ICIR、年化与 t 统计

FactorMiner 现在报告**四种**不同的量:

```text
ICIR                 = mean(IC_t) / sample_std(IC_t)
annualized ICIR      = ICIR * sqrt(periods_per_year)
independence t-stat  = ICIR * sqrt(number_of_periods)
HAC t-stat           = mean(IC_t) / NeweyWestSE(mean(IC_t))
```

**不要按字段名盲比**。[AlphaBench 的指标代码](https://github.com/CityU-MLO/AlphaBench/blob/main/backtest/factor_metrics/metrics.py)把 `FL_Ir` 定义为 `mean/std`,把 `FL_Icir` 定义为 `sqrt(n)*mean/std`。后者是**独立性 t 统计**,不是 FactorMiner 的未年化 ICIR。正的序列相关可以让那个独立性统计**显著过于乐观**;在新报告中,**HAC 统计是主要的渐近推断**。

### 符号与幅值

`mean(abs(IC_t))` 不是稳定方向的证据。一个 `+0.1, -0.1` 交替的序列有**很高的平均幅值**,但**零**可用的符号均值。FactorMiner 的论文门槛仍是 `abs(mean(IC_t))`,带符号的均值仅保留用于方向参考。**不要在持仓测试面板上选择因子的符号**;在训练数据上设置方向并**冻结**。

### 换手与成本

证据报告构造等权多头与空头两腿(每个单位名义),然后计算:

```text
one_way_turnover_t = 0.5 * sum_i(abs(w[i,t] - w[i,t-1]))
net_return_t       = gross_return_t - one_way_cost * sum_i(abs(delta_weight[i,t]))
```

它使用**实际的多空目标权重**,而不只是 top 桶的成员 churn。它**排除**初始入场并说明这一点。线性 basis-point 曲线是**可复现的压力测试**,不是撮合模拟器。当提供美元成交量时,既有的平方根冲击/容量估计器会加入参与度与资本场景。价差、费用、借入、涨跌停、延迟、队列与市场特有的税费假设**仍**需要生产级执行模型。AQR 关于[资产定价异常的交易成本](https://www.aqr.com/insights/research/working-paper/trading-costs-of-asset-pricing-anomalies)研究(基于大型实盘机构交易样本)是一个有用的提醒:**成本与盈亏平衡容量**是策略、市场、规模与执行特有的;泛化的 basis-point 折减仅是**第一道筛子**。

FactorMiner 当前的容量估计器使用**所选多头腿的平均流动性**为两腿参数化一个共享冲击折减。只有在多头与空头流动性相似时,这才是一个有用的诊断场景。**生产研究必须对两腿分别估计每笔交易**,包括借入可用性与非对称市场约束。

### 多重检验

试验数量**包括**被拒绝的公式、prompt 变体、模型变体、种子、超参数、检查过的宇宙与人工重启——**不**仅是保存的因子。[Harvey, Liu 与 Zhu 的 factor-zoo 分析](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=2249314)解释了为什么在大量搜索后,单纯的 `t > 2` **不再是**可信的发现门槛。

FactorMiner 使用**互补**的控制:

- 族内 Benjamini-Hochberg FDR(双侧 HAC 均值检验);
- 块自助 IC 置信区间;
- Deflated Sharpe(DSR),带声明的试验数与非正态收益矩;
- PBO 来自完整的"试验 × CPCV 路径"性能矩阵。

**FDR** 问的是"族内预期有多少发现是假的"。**DSR** 问的是"所选收益序列是否通过多重搜索/非正态门槛"。**PBO** 问的是"选择规则是否反复选出一个 in-sample 赢家,但落在 out-of-sample 中位数之下"。它们**不互替**。请见[原始 DSR 论文](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=2460551)与[PBO 论文](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=2326253)。Benjamini-Hochberg 对独立或正相关检验最站得住脚;**高度相关**的因子族还应考虑更保守的 Benjamini-Yekutieli 程序或基于重采样的族检验;当前实现报告这一限制,而不是声称"任意依赖下的控制"。

### 风险模型残差化

一个强因子可能是**伪装的大小、beta、动量、波动率、国家或行业**持仓。开源的残差化原语在每个日期独立拟合:

```text
signal_t = intercept + exposure_t * beta_t + residual_signal_t
```

它接受静态或点-in-time 数值敞口、one-hot 分类与可选的正 WLS 权重。然后**在残差信号上重做**指标与换手/成本压力。这实现的是**归因问题**,**不**是 MSCI 的专有模型构造。Barra 模型还会估计并标准化敞口、因子收益、因子协方差与特异风险;[MSCI USE4 方法论](https://www.msci.com/documents/10199/242721/Barra_US_Equity_Model_USE4.pdf/d7625289-cade-4e88-96ae-696219af5b67)说明了一个完整的商业风险模型是**更大得多的系统**。

## 公开 API

公开 API 是**累加式**的:

```python
from factorminer.evaluation import (
    IndustryEvidenceConfig,
    evaluate_industry_evidence,
)

report = evaluate_industry_evidence(
    "candidate_042",
    signals,                 # (assets, periods)
    forward_returns,         # 已经对齐 (assets, periods)
    config=IndustryEvidenceConfig(
        periods_per_year=252,
        cost_bps=(0, 5, 10, 20),
        primary_cost_bps=10,
    ),
    risk_exposures=style_and_industry_exposures,  # (M,K) 或 (M,T,K)
    risk_weights=float_market_cap,                # 可选 (M,) 或 (M,T)
    exposure_names=["size", "beta", "momentum", "industry_software"],
    family_ic_series=every_tried_factor_rankic,
    n_trials=total_search_trials,
    pbo_performance_matrix=trial_by_cpcv_path_scores,
    volume=point_in_time_dollar_volume,
)

payload = report.to_dict()  # 严格 JSON 兼容;NaN/inf 变为 null
```

结果包括:

- 精确的 Pearson IC 与 RankIC 序列与摘要;
- 未年化/年化 ICIR、独立性 t 统计与 HAC 推断;
- 块自助 RankIC 置信区间;
- 目标权重换手、毛收益序列、成本曲线与绝对价差盈亏平衡成本(测试集符号不翻转);
- 原始 vs 风格/行业残差化后的指标与成本压力;
- 可选的族 FDR、DSR、PBO 与平方根容量输出;
- 验证覆盖与对"数组无法建立的证据"的警告。

标准运行时评估器也会输出显式 Pearson 与 RankIC 字段。**既有 `ic_*` 消费者保留** Spearman RankIC 行为与 `paper_ic_v2` 版本,避免对历史分值的静默重写。
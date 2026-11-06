# 可复现性与基准语义

本文档定义 FactorMiner 能复现什么、输入面板如何被规范化、哪些指标控制准入与排序,以及基准来源该如何被解读。它是**技术契约**,不是性能声明。

## 复现级别

| 级别 | 含义 |
| --- | --- |
| Smoke | 在 mock 或内嵌样本数据上的确定性本地执行 |
| Repository-local 基准 | 带 manifest 的运行时重算,可能用部分/代理基线 |
| 论文协议运行 | 在用户提供的数据上,使用论文的目标/划分/选择/指标语义 |
| 论文级复现 | 等价数据/宇宙 + 忠实的外部基线 + 可比硬件 |

仓库支持**前三个**级别。它**不**打包论文的 A 股面板、64 资产 Binance 面板、外部基线实现,也不打包第四级所需的 A100 度量。

## 确定性本地检查

这些命令**不需要** API key 或私有数据集:

```bash
uv run factorminer quickstart
uv run python scripts/run_demo.py
uv run factorminer -o /tmp/factorminer-mock mine --mock -n 2 -b 8 -t 2
uv run factorminer --cpu benchmark table1 --mock --baseline factor_miner
```

`quickstart` 跑一次健康检查,挖出小型 mock 库,并在 `/tmp/factorminer-quickstart/` 下产出静态报告。Mock 结果**只证明执行与产物形状**;它们不是市场表现的证据。

## 市场数据契约

规范化的长格式面板要求:

```text
datetime, asset_id, open, high, low, close, volume, amount
```

接受的资产标识包括 `asset_id`、`code`、`ticker`、`symbol`、`ts_code`;`amt` 作为 `amount` 的别名。加载器会按资产与时间排序、校验数值,并可推导:

- `vwap = amount / volume`(volume 有效时);
- 每个资产的 `returns`(自 close 推导)。

这些**推导值是运行时变换**,不能当作 vendor 观测字段。挖掘前应跑严格校验:

```bash
uv run factorminer validate-data path/to/market_data.csv --strict
```

MCP 连接器配置必须**映射或显式推导**每个必需字段。连接器 schema 与信任边界见 [Factor Researcher 集成](../integrations/factor-researcher/README.md)。

## 内嵌的 Binance 形态样本

`data/binance_crypto_5m.csv` 是一个 onboarding 与 CLI-smoke 样本,**不是**论文的加密数据集。它的机器可读来源信息在 `data/binance_crypto_5m.manifest.json`;描述性说明在 [`data/README.md`](../data/README.md)。

样本包含 20,000 行 5 分钟数据:20 个标的,每个 1,000 行,从 `2026-02-14 21:30:00` 到 `2026-02-18 08:45:00`。请用配套的短窗口 profile 以避免空论文周期划分:

```bash
uv run factorminer --config factorminer/configs/binance_sample.yaml \
  validate-data data/binance_crypto_5m.csv
```

论文风格的 Binance 通道使用 10 分钟 bar。Binance spot klines 没有原生的 `10m` 间隔,因此**重采样** 1 分钟或 5 分钟的源 bar:

```bash
uv run factorminer resample-data \
  data/binance_crypto_5m.csv \
  /tmp/binance_crypto_10m.csv \
  --rule 10min
```

聚合是**确定性的**:

| 字段 | 聚合 |
| --- | --- |
| `open` | 首个源 bar |
| `high` | 最大 |
| `low` | 最小 |
| `close` | 末个源 bar |
| `volume`、`amount` | 求和 |
| `vwap` | 重算为重采样后的 `amount / volume` |
| `returns` | 按资产从重采样 close 重算 |

对于用户提供**完整**的加密面板,从 `factorminer/configs/paper_repro_binance.yaml` 开始。它声明 Binance/加密、10 分钟频率、2024 训练区间与 2025 持仓区间。

## 目标与划分语义

面向论文的张量契约支持**下期 open-to-close 目标**。运行时划分边界来自 `data.train_period` 与 `data.test_period`。`evaluate`、`combine`、`visualize` 与 `benchmark.runtime` 共享**同一** `EvaluationDataset` 划分模型。

**不要**用测试期证据做候选准入、模型拟合、因子权重拟合或超参数选择。暴露显式 fit 与 eval 周期的命令应保留 `train → test` 方向。

## 信息系数(IC)语义

在每个时间步,FactorMiner 历史 `ic_*` 字段计算**因子信号与未来收益**之间的**截面 Spearman 秩相关**。在行业术语中,这是 **RankIC**,**不**是 Pearson IC:

```text
legacy IC_t = RankIC_t = SpearmanRankCorr(signal_t, forward_return_t)
```

摘要字段含义如下:

| 字段 | 定义 | 用途 |
| --- | --- | --- |
| `ic_mean` | `mean(IC_t)` | 符号感知诊断与权重 |
| `ic_paper_mean` | `abs(mean(IC_t))` | 论文模式准入、替换、排序与 Top-K 冻结 |
| `ic_abs_mean` | `mean(abs(IC_t))` | 仅作为遗留的不稳定性诊断 |
| `icir` | `mean(IC_t) / std(IC_t)` | 符号化诊断 |
| `ic_paper_icir` | `abs(mean(IC_t)) / std(IC_t)` | 论文模式 ICIR 门槛与报告 |

新评估产物同时暴露 `rank_ic_*` 与 `pearson_ic_*` 字段,以及 `ic_definition: spearman_rank`。遗留字段保持不变,以避免老的库与准入门槛被默默重新解释。行业证据协议额外区分:未年化的 ICIR、年化的 ICIR、独立性 t 统计与 Newey-West/HAC t 统计。精确公式与跨平台对比规则见 [Evidence Protocol(中文)](evidence-protocol-中文.md)。

**区别是实质性的**。对 `IC_t = [0.1, -0.1]`,`ic_abs_mean = 0.1`,而 `ic_paper_mean = 0.0`;该序列有幅值但**没有稳定的平均方向**。论文模式门槛使用后者。

带显式字段的产物携带 `metric_version: paper_ic_v2`。老库以兼容性默认值读取并被标记为 `legacy_abs_ic`;在与当前输出对比前请**重跑评估**。

## 运行时重算

保存的库分值**不是**分析的权威输入。给定一个公式与数据集,运行时解析器与算子注册表会**重新计算**信号、划分统计与组合诊断。这可以防止过期元数据被报告为**不同面板**上的证据。

示例:

```bash
uv run factorminer --cpu evaluate output/factor_library.json \
  --data path/to/market_data.csv --period both --top-k 10
```

当公式使用了不可用的叶子、不支持的算子、无效窗口或不兼容的面板时,重算会**显式失败**。失败的解析或评估**绝不能**用保存的分值替代。

## 基线来源

每个基准 manifest 标识其基线如何获得。

| 来源 | 含义 |
| --- | --- |
| `builtin_paper_catalog` | 内置的 110 个规范化 FactorMiner 公式 |
| `saved_library` | 用户提供的 `FactorLibrary` |
| `runtime_loop` | 在基准运行中实时挖掘的因子 |
| `catalog_baseline` | 命名公式家族的确定性本地近似 |
| `catalog_proxy` | 未在此捆绑的外部算法的占位 |
| `synthetic_no_memory_proxy` | 没有真实无记忆运行时的兜底近似 |

当前 CLI 基线保真度:

| 基线 | 实现状态 |
| --- | --- |
| `factor_miner` | 内置目录、保存库或运行时循环 |
| `factor_miner_no_memory` | 仅在有真实 runtime/saved 无记忆库时忠实;否则标为 synthetic proxy |
| `alpha101_classic` | 确定性的部分目录,**不**含全部 101 个公式 |
| `alpha101_adapted` | 带本地窗口变体的部分目录 |
| `random_exploration` | 有界的确定性模板,**不**是论文的完整随机树实现 |
| `gplearn` | catalog proxy,**不**是外部包 |
| `alphaforge_style`、`alphaagent_style` | catalog 派生的 proxy |

只有当算法/公式、生成与选择规则、数据集/宇宙、训练/测试协议、指标版本**全部对齐**时,一个命名基线的结果才算论文级。否则请称之为 repository-local 基准、catalog baseline、smoke 基准或 proxy。

## 规范基准执行

`factorminer.benchmark.runtime` 协调基准计算。契约构造、来源捕获、循环执行、数据集、冻结评估、统计、报告都在它们命名的基准模块中。CLI 与独立报告构建器委托给它:

```bash
uv run factorminer -c factorminer.local.yaml -o output-benchmark \
  benchmark table1 --data path/to/market_data.csv --baseline factor_miner

uv run factorminer --cpu benchmark ablation-memory --mock
uv run factorminer --cpu benchmark ablation-strategy --mock
uv run factorminer --cpu benchmark cpcv --mock
uv run python scripts/run_phase2_benchmark.py --mock
```

基准产物包括:指标版本、所选基线与类型、候选数量、数据集身份/哈希、运行时契约以及对 proxy/兜底行为的警告。**JSON 产物和它们的 manifest 必须放在一起**;没有 manifest 的渲染表格**不是**可复现的基准。

## 论文声明覆盖

| 表面 | 状态 |
| --- | --- |
| 类型化 OHLCV 公式 DSL 与论文风格算子别名 | 已实现 |
| Ralph 生成/评估/进化循环与结构化记忆 | 已实现 |
| IC、冗余、替换、去重与全验证门槛 | 已实现 |
| 110 个规范化的内置论文因子 | 已实现 |
| 下期论文风格目标与可配置训练/测试契约 | 已实现 |
| 精确的 A 股与 64 资产 Binance 论文数据集 | 用户提供;不打包 |
| 忠实复现的完整 Alpha101/GPLearn/AlphaForge/AlphaAgent 基线 | 不打包;按标注为部分或 proxy |
| A100 速度表与论文级效率数据 | 不打包 |

NumPy、由 Bottleneck 加速的 CPU 与 torch/GPU 执行面可在本地基准化,但结果描述的是**实际主机与代码路径**。不要把本地计时说成论文的硬件结果。

## 复现清单

针对送审的结果,应保留:

1. 源数据的精确身份、许可证/授权、抓取时间戳与内容哈希;
2. 规范化/重采样步骤,以及是否有任何字段是派生的;
3. 配置、代码 commit、Python/包环境、后端、随机种子;
4. 训练/测试边界、目标构造、宇宙成员、成本;
5. 因子库 + 指标 + 基线来源;
6. 机器可读的结果/manifest 文件,而不是只有截图或 Markdown;
7. 警告、被拒绝的候选、重算失败;
8. 在相同输入上**独立 replay 后**再做对比声明。
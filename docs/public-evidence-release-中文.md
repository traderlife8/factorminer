# 公开证据发布 Runbook

本 runbook 把公开数据实验变成一个**独立可验证、内容寻址**的发布。它分开三种**不能混淆**的声明:

1. 提供商档案以**committed lock 中记录的 URL 与 SHA-256** 被抓取;
2. FactorMiner 在**冻结的配置与种子**下对**那些精确字节**做了规范化与评估;
3. 产出的 receipt 与每个声明的产物**仍**与它们记录的哈希匹配。

它**不**声明历史结果会持续、所选宇宙是点-in-time、或原始提供商数据可被再分发。

## 参考数据集

发布安全的参考覆盖 **12 个 ECB 日度欧元外汇参考汇率系列**,时间从 2023-01-01 到 2024-12-31。**只要**信息被准确复述、注明 ECB、且声明修改,ECB 允许免费复用其站点信息。可编辑的 YAML 冻结**宇宙、变换与限制**;JSON lock 冻结**每个 API 响应的精确 SHA-256**。

ECB 表示这些"仅信息"的参考观察一般在工作日约 16:00 CET 发布。规范化的午夜时间戳**只是观察日标签**:流水线把该值视为**该发布后**可用,并**仅**评估**随后的日度观察**。

```bash
uv sync --python 3.12 --extra dev --extra visualization

uv run factorminer public-data prepare \
  examples/public_evidence/ecb_fx_daily_2023_2024.lock.json \
  output/public-evidence-data

uv run factorminer public-data verify output/public-evidence-data
```

预期的规范面板有 **6,132 行、12 个资产、511 个日度时间戳**,SHA-256 为
`d9a17748808f29892901fae6c1ef37bfba3bdaca06df703c943d954afe5d1096`。
committed lock 的 SHA-256 是
`9c5c4630ee789bf7e78ce192cd8e05bf349756247ef55aaa312227ca7bf9a0b6`。
**不匹配就是新数据集版本**,而不是同一实验。

在不改动 committed lock 的情况下审计提供商校验和解析:

```bash
uv run factorminer public-data lock \
  examples/public_evidence/ecb_fx_daily_2023_2024.yaml \
  output/recreated-public-data.lock.json
cmp examples/public_evidence/ecb_fx_daily_2023_2024.lock.json \
  output/recreated-public-data.lock.json
```

## 全证据运行

跑三个**无需凭证**的公式基线、十个派生种子与 Tier-0 证据报告。Ralph/Helix 生成**需要**外部模型提供商,**排除**在 fresh-clone 声明外。规范化的 ECB 面板在其声明的复用条件与归属下打包。

runner **确定性地**把唯一时间戳划分为 60% 训练、20% 验证、20% 测试窗口。在每个边界**purge** 一个时间戳,使 fit/selection 窗口末端的"下一 bar"目标**不能**进入下一个窗口;记录的 embargo 为**零 bar**。**准入**在 train 上拟合,**Top-K 选择与组合权重**使用 validation,**Tier-0 报告**仅使用 test。

```bash
uv run python scripts/run_phase2_benchmark.py \
  --data output/public-evidence-data/market_data.csv \
  --dataset-manifest output/public-evidence-data/dataset_manifest.json \
  --data-license-class redistributable_with_attribution \
  --evidence-tier public_reproducible \
  --portable-release \
  --bundle-public-data \
  --runs 10 \
  --seed 42 \
  --n-factors 40 \
  --skip-ablation \
  --output output/public-evidence-run \
  --methods random_exploration alpha101_classic alpha101_adapted
```

输出包括:**每个种子的运行时 manifest、基线对比表、Pearson IC、RankIC、HAC 不确定性、换手/成本压力路径、DSR 与显式的 missing-gate 状态、统计检验、有效配置、报告**以及 `output/public-evidence-run/releases/` 下的内容寻址目录。

## Fresh-clone 验证

在另一台机器上,checkout receipt 的 `code_sha`,从 committed lock **重建数据**,然后跑:

```bash
uv sync --python 3.12 --frozen --extra dev --extra visualization
uv run --python 3.12 factorminer verify-receipt \
  output/public-evidence-run/releases/RELEASE_ID
```

变更的 receipt、manifest、声明的产物、环境锁或数据集承诺都会让验证**失败**。**可重定位**的产物路径意味着发布目录**可以**被复制,**无需**保留生产者的绝对文件系统布局。

## 发布清单

- 把**可移植发布目录**作为归档附上,在其数据集描述符中保留 ECB 归属与变换披露;
- 在 GitHub Release 笔记中记录**仓库 commit 与 release ID**;
- 报告**每一种方法与种子**,而**不**仅是最好的运行;
- 保留 manifest 的**基线来源**。仓库的 `alpha101_classic` 与 `alpha101_adapted` 目录是**版本化的已实现子集/proxy**,**不能**被描述为完整的 Alpha101、Alpha158、Alpha191 或 Alpha360 实现;
- 声明参考观察**不是**可执行收盘、OHLC 字段**派生自一个观察**、成交量是 schema 占位符、流动性与容量**不可用**、宇宙是**事后选择**的、**没有**结果是实盘证据;
- 链接到**精确的 lock 与本 runbook**;
- 在称之为"独立复现"前,由**独立的人**遵循 fresh-clone 流程。

源系列通过 [ECB Data Portal](https://data.ecb.europa.eu/data/datasets/EXR) 发布;复用受 ECB 的[免责声明与版权条件](https://www.ecb.europa.eu/services/using-our-site/disclaimer/html/index.en.html)约束。发布时机与"仅信息/交易"警告来自 [ECB 参考汇率页面](https://www.ecb.europa.eu/stats/policy_and_exchange_rates/euro_reference_exchange_rates/html/index.en.html)。

面板**显式**声明 FactorMiner **反转了汇率**并**派生**了 schema 字段。单独的 Binance 示例保留**真实**的价/量字段用于技术对比,但**不是**可再分发的发布输入。
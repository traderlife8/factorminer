# 贡献指南

FactorMiner 的变更必须保留**研究协议、运行时重算、产物来源与包归属边界**。本文档定义**仓库工作流**;具体技术行为由对应文档承载。

## 环境搭建

需要 **Python 3.12 或更高版本**。CI 与仓库工具在 Python 3.12 上运行,这是贡献者与发布版本的兼容性基线。

```bash
git clone https://github.com/minihellboy/factorminer.git
cd factorminer
uv sync --group dev --all-extras
```

精简环境可用 `uv sync --group dev`,按需追加 `--extra llm` 或 `--extra mcp`。请通过 `uv run` 跑仓库命令。

## 分支与 PR

从**已更新**的 `main` 开新分支:

```bash
git switch main
git pull --ff-only
git switch -c <type>/<descriptive-scope>
```

使用语义化的名称,例如 `fix/ic-split-leakage`、`refactor/cli-command-registration`、`docs/repository-structure`。**不要**在分支名后追加工具名、编辑器或作者后缀。

每个 PR 应**独立可审查**:

- 一个行为契约**或**一次纯机械重组;
- 聚焦的、能证明变更的测试;
- 同步更新**拥有该行为的文档**;
- 不要混入不相关的格式或生成的运行时输出;
- 跑通 lint、manifest、测试与打包检查。

优先**连续的小 PR**,而不是一个混合了文件移动、语义变更、新功能与清理的分支。纯粹的兼容性移除必须**指明迁移的使用方**与**证明旧路径已不再需要**的测试。

## 包归属

| 路径 | 职责 |
| --- | --- |
| `factorminer/domain/` | 无外部依赖的、跨应用工作流共享的数值契约 |
| `factorminer/application/` | 类型化执行上下文与跨工作流的应用契约 |
| `factorminer/architecture/` | 协议契约、策略、阶段、可复用的研究服务 |
| `factorminer/core/` | 循环编排、DSL / 解析器、表达式、因子库、会话 I/O |
| `factorminer/agent/` | 模型提供商、prompt 构造、生成、辩论 |
| `factorminer/data/` | 摄取、规范化、连接器、张量构建 |
| `factorminer/evaluation/` | 运行时重算、指标、校验、报告 |
| `factorminer/benchmark/` | 对比契约、数据集、运行器、统计与报告 |
| `factorminer/memory/` | 存储、检索原语、知识图谱、嵌入 |
| `factorminer/operators/` | 类型化算子规格与执行后端 |
| `factorminer/mcp/` | 外部 MCP 工具/资源,封装在稳定的引擎工作流之上 |
| `integrations/` | 可部署的智能体打包;**不**复制引擎逻辑 |
| `scripts/` | 仓库校验、演示、独立入口 |

依赖方向与运行时契约见 [Architecture(中文)](docs/architecture-中文.md)。

## 工程规则

1. **面向基准的语义**放在架构契约或 `factorminer.benchmark.runtime` 上,**不要**写在 CLI 处理器或单一循环里。
2. 在所提供的数据集上**重算**公式信号。**保存的摘要分值是元数据**,**不是**权威的分析输入。
3. 把**检索、演化与持久化行为**封装在 `MemoryPolicy` 后面。
4. 优先**可复用的服务或阶段实现**,而不是给 `RalphLoop` / `HelixLoop` 加分支。
5. 通过 `FactorAdmissionService` 修改因子库,使准入与替换不变量**集中在单一职责**。
6. 可选实验保持**opt-in 且对准入中性**,除非有**显式且经测试**的契约变更。
7. `output/`、凭证、私有数据与本地配置**不进源码控制**。
8. 为**每个新公共契约、策略、阶段、基准路径、manifest 字段或公共导出**添加回归覆盖。
9. 所有仓库作者贡献的源码、文档与资产均以 **MIT 许可证**发布。贡献即代表你同意你的贡献以 MIT 许可证发布。**不要**引入条款不允许以 MIT 发布的代码或资产。

## 文档归属

技术文档**只描述已实现的行为**:

- `README.md`:**产品边界**、支持的表面、设置、常用命令与仓库地图;
- `docs/architecture.md`:契约、流程、不变量与包归属;
- `docs/reproducibility.md`:数据、划分、指标、基线来源与基准解读;
- `docs/security.md`:信任边界、控制项与验证;
- `integrations/<name>/README.md`:集成特定的 manifest、接口、部署要求与权限。

**不要**把路线图、审计日志、推测性功能、市场预测或已完成的工作清单放进技术文档。Git 历史、Issue 与 PR 才是规划与完成工作的记录载体。

## 校验

编辑时跑**快速、聚焦**的检查:

```bash
uv run ruff check <changed-python-files>
uv run pytest -q path/to/relevant_test.py
```

提交 PR 前跑:

```bash
uv run ruff check .
uv run python scripts/check_architecture.py
uv run python scripts/check.py
uv run pytest -q factorminer/tests
uv build
```

实用的聚焦测试集:

| 变更 | 最低聚焦覆盖 |
| --- | --- |
| 架构契约 | `factorminer/tests/test_architecture.py` |
| Ralph / Helix 生成或阶段 | `test_ralph_loop.py`、`test_helix_loop.py` 与生成器测试 |
| 基准运行时或独立脚本 | `factorminer/tests/test_benchmark.py` |
| MCP / 集成 | MCP 测试 + `uv run python scripts/check.py` |
| 包导出 / 依赖 | `factorminer/tests/test_import_boundaries.py` |

提交前使用 `git diff --check`。构建产物**必须**包含 `factorminer` 包与 `configs`,**不应**包含测试、本地输出、文档归档、集成模板或私有数据——除非打包元数据**显式**另有声明。

## PR 描述

审查者应能回答:

- 改的是哪个**契约或路径**?
- 影响哪些**执行表面**?
- 哪些**公共导入或产物**保持稳定?
- 哪些测试证明结果?
- 哪份**技术文档**被同步更新?
- 是否涉及**数据、安全、迁移或可复现性**的影响?
# 托管 Pilot 安全与运维

托管 pilot 是一项**持久、租户隔离**的研究作业服务。它**不是**本地 `mcp-serve` 命令,**不**暴露任意命令,**绝不**下单。它的安全表面被有意做得小:**校验数据**、**挖掘有界因子库**、**跑绑定数据集的基准**、**生成报告**与**校验 receipt**。

## 威胁模型

| 威胁 | 强制控制 | 残留风险 / 操作员职责 |
| --- | --- | --- |
| 跨租户读 | 租户 ID 来自已认证主体;DB 查询与路径再次绑定它 | 每个新工具后**测试授权** |
| 路径穿越或 symlink 逃逸 | 相对路径规范化、租户根路径 containment、作业前后 symlink 拒绝 | 在生产中以 OS/容器级别隔离 worker |
| 命令注入 | 类型化作业 schema 与固定 argv;无 shell;最小环境 | 依赖仍处理不可信数据 |
| 资源耗尽 | 请求、队列、活跃作业、运行时、输入、日志、总存储与日计算配额 | 加上基础设施 CPU/内存/进程/网络限制 |
| 丢失/重放请求 | 幂等键、SQLite WAL 持久化、作业租约与崩溃恢复 | 一致地备份 DB 与存储 |
| Token 或提供商密钥被窃 | 高熵哈希的租户 token;提供商密钥**仅**经显式 worker 白名单转发,**绝不**序列化 | 使用密钥管理器、短生命周期提供商凭证与出站策略 |
| 审计篡改 | 租户特定的 append-only 哈希链 | 把审计导出复制到不可变的外部存储 |
| 学习数据泄露 | 显式目的同意、固定指标/上下文、撤销删除、按租户均值、k 匿名 | 多次发布可允许差分攻击;人工审查快照 |
| 过期产物 | 租户保留期、dry-run 清理、精确的租户删除确认 | 调度与监控执行 |

SQLite 在**单一主机**上提供持久的 pilot 协调,**不是**水平扩展的共识。**不要**让多台机器对网络挂载的 SQLite 文件操作。

每次重试写到 attempt-scoped 的作业目录。如果崩溃的 worker 留下孤儿子进程,**恢复的 attempt 不能损坏或替换其产物**。进程监管器或 OS/容器边界**仍**必须收割孤儿进程组并强制 CPU/内存限制。

请求速率检查与日计算配额与接收对应事件或作业的**同一数据库事务**一起序列化,**防止**并行提交绕过这些应用层配额。保留每次调用移除**最旧** 1,000 个符合条件的作业目录;重复调度该命令,**直到** dry run 为空。

## 认证边界

Pilot 凭证是**操作员签发的不透明 bearer token**。每个 token 在静态时**哈希**,并携带显式作用域:

- `jobs:submit`、`jobs:read`、`jobs:cancel`
- `artifacts:read`、`usage:read`、`consent:write`

托管 MCP 服务器在**每一次工具调用**时对 DB 重新校验 token,并把 SDK access-token 资源与客户端 ID 绑定到配置的 MCP 资源与租户。**非 loopback 启动**需要 HTTPS 资源 URL。

生产环境应在资源服务器前置一个**符合标准**的 OAuth 授权服务器,验证 resource audience,发布受保护资源元数据,并请求**最小特权**作用域。MCP [授权规范](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)定义了这些要求。Python SDK 依赖保持在稳定的 1.x(`mcp>=1.27,<2`),因为 2.x 尚不稳定。

## 配置与服务启动

```bash
uv sync --extra mcp

uv run factorminer hosted-pilot init \
  --db /srv/factorminer/control.sqlite3 \
  --storage-root /srv/factorminer/tenants

uv run factorminer hosted-pilot tenant-create partner-01 \
  --display-name "Partner 01" \
  --retention-days 30 \
  --db /srv/factorminer/control.sqlite3 \
  --storage-root /srv/factorminer/tenants

uv run factorminer hosted-pilot credential-create partner-01 \
  --label pilot-client \
  --scope jobs:submit --scope jobs:read --scope jobs:cancel \
  --scope artifacts:read --scope usage:read --scope consent:write \
  --expires-days 30 \
  --db /srv/factorminer/control.sqlite3 \
  --storage-root /srv/factorminer/tenants
```

凭证**只打印一次**。直接放入客户端**经批准**的密钥库。**通过操作员命令**输入资源;远程 MCP 调用者**不能**命名服务端源路径:

```bash
uv run factorminer hosted-pilot put-input partner-01 reviewed-market.csv \
  --name market.csv \
  --db /srv/factorminer/control.sqlite3 \
  --storage-root /srv/factorminer/tenants
```

在进程监管器下运行至少一个 worker 与无状态 JSON MCP 资源服务器:

```bash
uv run factorminer hosted-pilot worker \
  --worker-id worker-01 \
  --db /srv/factorminer/control.sqlite3 \
  --storage-root /srv/factorminer/tenants

uv run factorminer hosted-pilot serve \
  --issuer-url https://auth.example/ \
  --resource-url https://pilot.example/mcp \
  --host 127.0.0.1 --port 8766 \
  --db /srv/factorminer/control.sqlite3 \
  --storage-root /srv/factorminer/tenants
```

在维护的反向代理处终止 TLS,在那里限制请求体大小与连接速率,并**仅暴露**托管路径。**永远不要**把本地单操作员 MCP 表面路由到租户面向的监听器。

挖掘作业需要**配置好的模型提供商**。默认情况下 worker **不继承**任何提供商密钥。如果 pilot 需要,操作员可以显式加入(例如 `--provider-secret-env GOOGLE_API_KEY`);**仅接受**三个文档化的 provider-key 名称,密钥值**保留**在 worker 环境中,而非作业参数、日志、DB 行或产物。优先使用由进程监管器注入的**短生命周期密钥**。

## 运维控制

在接受一个 partner 前:

- 给每个租户**唯一** ID 与存储命名空间;
- 在可能时把默认配额**降到**约定的工作量;
- 签发**最小**的作用域集合与**最短**的有效期;
- 在部署镜像里跑跨租户与路径穿越测试;
- 配置 OS 级别的 CPU、内存、进程、磁盘与出站限制;
- 加密磁盘与备份,并把密钥访问日志记在本应用**之外**;
- 指定服务所有者与事件响应代理,并在部署的私有运维系统中保存**当前**联系方式与升级细节。

日常运维:

```bash
uv run factorminer hosted-pilot audit-verify partner-01 DB_AND_STORAGE_OPTIONS
uv run factorminer hosted-pilot retention-purge partner-01 DB_AND_STORAGE_OPTIONS
uv run factorminer hosted-pilot retention-purge partner-01 --apply DB_AND_STORAGE_OPTIONS
uv run factorminer hosted-pilot usage-export partner-01 usage.jsonl DB_AND_STORAGE_OPTIONS
```

把 `DB_AND_STORAGE_OPTIONS` 替换为相同的显式 `--db` 与 `--storage-root`。**第一个**保留命令是 dry run。在**同一一致性点**备份 SQLite DB 与租户根,测试恢复,并把验证过的审计/用量导出复制到**不可变**存储。

怀疑 token 泄露时,**吊销**该凭证,**签发**新的,**检查**租户审计链与用量,**取消**可疑作业,并保留 DB 与日志。要完整抹除时,**先**导出策略所需的证据,然后要求**精确确认**:

```bash
uv run factorminer hosted-pilot tenant-delete partner-01 \
  --confirm partner-01 DB_AND_STORAGE_OPTIONS
```

租户记录、凭证、观察项与文件被删除。**仅**保留一个**化名**的删除墓碑,用于审计连续性。

## 基于同意的学习

学习**默认关闭**,直到租户**显式授予**一个固定目的。**仅接受**白名单中的数值指标与粗粒度上下文;**不接受**自由文本、公式、仓位、收益、prompt 与租户名。撤销会**立即删除**该租户针对该目的的观察项。

快照把每个观察项绑定到**活动**的同意策略版本,**先**对每个租户取均值以防高量租户占主导,然后**压制**低于阈值的组。**内容衍生的**快照 ID 为该聚合打上版本号。操作员命令默认为**五个不同**租户:

```bash
uv run factorminer hosted-pilot consensus-snapshot \
  aggregate_workflow_metrics aggregate-workflow.json \
  --min-tenants 5 DB_AND_STORAGE_OPTIONS
```

每个发布的快照都需要对小群体、相对早期快照的差分、稀有上下文组合以及"同意策略版本是否仍描述预期用途"做**人工隐私审查**。
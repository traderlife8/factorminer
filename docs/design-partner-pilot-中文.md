# Design-Partner Pilot 协议

一个 design-partner 结果**只有**在**真实外部审阅者控制输入、观察协议、并选择签署哪些有界声明**时才可信。本项目**不能自我签发**该确认。

## 角色与证据边界

| 角色 | 控制 | 可断言 |
| --- | --- | --- |
| 生产者 | 代码、配置、可移植产物 | 软件运行了什么 |
| Partner | 专有数据集、审阅密钥、发布同意 | 仅限**亲自观察**的行为 |
| 验证者 | receipt、请求、确认、公开的验证密钥 | 完整性与精确的声明绑定 |

私有 receipt 存储**HMAC-SHA-256** 数据集承诺,而**不是**数据集或密钥。对数据集承诺与审阅确认使用**不同的随机密钥**。两者都**保持在 Git、shell 历史、报告与可移植产物之外**。

## 1. 跑 partner 拥有的数据集

Partner 在其环境中跑命令,或通过经批准的临时密钥文件提供承诺密钥:

```bash
uv run python scripts/run_phase2_benchmark.py \
  --data PARTNER_MARKET_DATA.csv \
  --data-license-class proprietary_licensed \
  --evidence-tier private_partner_observed \
  --commitment-key-file PARTNER_DATASET_KEY.hex \
  --portable-release \
  --runs 10 \
  --seed 42 \
  --n-factors 40 \
  --full-ablation \
  --output output/partner-pilot
```

**没有任何原始 partner 数据被复制进发布**。

## 2. 准备有界的审阅请求

```bash
uv run factorminer partner-review prepare \
  output/partner-pilot/releases/RELEASE_ID \
  output/partner-pilot/review-request.json \
  --partner-pseudonym partner-01 \
  --assertion protocol_observed \
  --assertion dataset_commitment_verified \
  --assertion artifacts_reviewed \
  --assertion limitations_acknowledged \
  --assertion results_reproduced
```

只请求 partner **处于能检查的位置**的声明。**请求不是确认,也从不改变 receipt 的状态**。

## 3. Partner 记录结果

反馈文件**故意**是非叙事的,以减少意外泄露:

```json
{
  "setup_minutes": 30,
  "campaign_completed": true,
  "completion_rate": 1.0,
  "trust_rating": 4,
  "decision_outcome": "pilot_more",
  "adoption_blockers": ["data_integration", "security_review"],
  "repeated_use_intent": "yes",
  "useful_outputs": ["evidence_report", "receipt"],
  "failure_modes": ["none"],
  "requested_integrations": ["market_data", "sso"]
}
```

允许的 decision 值是 `adopt`、`pilot_more`、`not_now`、`reject` 与 `undisclosed`。blocker 取自**固定的分类法**。**名字、自由文本、仓位、收益、公式与专有元数据会被拒绝**。

然后 partner 选择**实际观察到的**断言:

```bash
uv run factorminer partner-review acknowledge \
  output/partner-pilot/review-request.json \
  output/partner-pilot/acknowledgment.json \
  --reviewer-pseudonym reviewer-01 \
  --assertion protocol_observed \
  --assertion artifacts_reviewed \
  --assertion limitations_acknowledged \
  --publication-consent anonymous \
  --key-file PARTNER_REVIEW_KEY.hex \
  --feedback-json PARTNER_FEEDBACK.json
```

`private` 同意把确认**保留为内部**;`anonymous` 允许**化名**记录;`public` 允许**声明的审阅者身份**。同意**不**扩大已签署的断言。

内置的 HMAC 对共享 partner 控制审阅密钥的各方证明完整性;它**不是**公钥身份证明,因为**任何密钥持有者都可以**产生签名。因此,公开具名背书**还**需要 partner 的常规批准发布渠道或**独立管理**的公钥签名。**不要**仅凭 `publication_consent: public` 把其本身变成更强的身份声明。

## 4. 验证并诚实报告

```bash
uv run factorminer partner-review verify \
  output/partner-pilot/review-request.json \
  output/partner-pilot/acknowledgment.json \
  output/partner-pilot/releases/RELEASE_ID \
  --key-file PARTNER_REVIEW_KEY.hex
```

验证把确认**绑定**到**精确的**审阅请求、receipt release ID、receipt 字节、断言子集与 HMAC 签名。任何编辑都会**使内容衍生的 ID 或签名失效**。**只在同意的级别上发布**,并把**部分完成或拒绝**作为证据报告,而**不是**默默丢弃不利的 pilot。
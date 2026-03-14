# 模型倍率配置与上架验证 — 端到端实施计划

> **For Claude:** 本方案为「摸清 new API 模型倍率设置完整流程 + 在一个模型上跑通并验证计费」的操作指南与思维框架，不要求使用 executing-plans 逐任务执行；可按文档自行实施并在下周内完成单模型验证。

**Goal:** 在关闭自用模式后，为至少一个模型配置倍率，完成「添加倍率 → 推理请求 → 核实 newAPI 扣费 → 核实 GCP 账单」的端到端验证（在自有 service account 上）。

**Architecture:** 倍率存储在 Option 表（key=ModelRatio），内存由 ratio_setting 维护；relay 用请求的模型名（OriginModelName）查倍率并计算预扣/结算 quota；GCP 侧用量在 Cloud Billing 中核对。选一个通道（建议 Vertex/Gemini 等自有 GCP 项目）上的一个模型做全链路验证。

**Tech Stack:** Go 后端（option/ratio_setting/relay/helper/price）、管理后台（Setting → Ratio / 模型定价）、Python 调用 OpenAI 兼容 API、GCP Console Billing。

---

## 一、new API 模型倍率流程梳理（思维框架）

### 1.1 概念对应关系

| 概念 | 说明 | 关键位置 |
|------|------|----------|
| **Channel** | 上游 API 通道（如 Vertex、Gemini、OpenAI），库表 `channels`，字段 `models` 为逗号分隔的模型名列表 | `model/channel.go` |
| **模型名** | 用户请求的 `model` 参数 = 计费用的 `OriginModelName`；可与上游名做映射（channel 的 model_mapping） | `relay/helper/model_mapped.go` |
| **倍率 (ratio)** | 按量计费：quota = tokens × model_ratio × completion_ratio × group_ratio 等；按次计费用 model_price | `setting/ratio_setting/model_ratio.go`, `relay/helper/price.go` |
| **自用模式** | `SelfUseModeEnabled=true` 时允许未配置倍率的模型；关闭后必须配置倍率或用户勾选「接受未设置倍率」 | `setting/operation_setting/operation_setting.go`, `controller/model.go` |

### 1.2 倍率数据流

```
[持久化]  Option 表 key="ModelRatio", value=JSON  {"model-a": 1.0, "model-b": 2.5}
    ↓ 启动/定时从 DB 加载 (model/option.go loadOptionsFromDatabase)
[内存]   ratio_setting.modelRatioMap (setting/ratio_setting/model_ratio.go)
    ↓ 请求时查询 GetModelRatio(originModelName)
[计费]   relay/helper/price.go ModelPriceHelper → PriceData.ModelRatio → 预扣/结算 quota
```

- **写入路径**：管理后台「设置 → 倍率/模型定价」编辑 ModelRatio → `PUT /api/option/` → `model.UpdateOption("ModelRatio", value)` → `ratio_setting.UpdateModelRatioByJSONString` 更新内存并写 DB。
- **默认值**：`setting/ratio_setting/model_ratio.go` 的 `defaultModelRatio`；重置接口 `POST /api/option/rest_model_ratio` 会恢复为该默认 JSON 并写库。

### 1.3 计费时用的模型名

- 普通 Chat/Completion：`OriginModelName` = 用户请求的 `model`（若 channel 有模型映射，则可能先做映射再查倍率；若为 ResponsesCompact 模式会带 compact 后缀）。
- **倍率表里的 key 必须与「计费时使用的模型名」一致**。建议先不启用复杂映射，选一个 channel 的 `models` 里明确写着的模型名，在倍率里配同名 key 即可。

### 1.4 关键代码索引

| 用途 | 文件 | 说明 |
|------|------|------|
| 倍率默认值/查询/更新 | `setting/ratio_setting/model_ratio.go` | `GetModelRatio`, `UpdateModelRatioByJSONString`, `defaultModelRatio` |
| 计费计算 | `relay/helper/price.go` | `ModelPriceHelper`（按量）, `ModelPriceHelperPerCall`（按次） |
| 未配置倍率时的错误 | `relay/helper/price.go` | 报错「模型 xxx 倍率或价格未配置…」除非 `AcceptUnsetRatioModel` 或自用模式 |
| Option 读写 | `model/option.go` | `UpdateOption`, case `"ModelRatio"` 调 `UpdateModelRatioByJSONString` |
| 定价 API/重置 | `controller/pricing.go` | `GetPricing`, `ResetModelRatio` |
| 模型列表与定价展示 | `model/pricing.go` | `updatePricing` 聚合 channel 能力并填 `model_ratio` / `model_price` |
| 同步倍率（可选） | `controller/ratio_sync.go` | 从 OpenRouter 等拉取并合并 model_ratio |

---

## 二、端到端实施步骤（单模型跑通）

### Task 1：确认环境与自用模式状态

**目的：** 确保当前为「关闭自用模式、需配置倍率」的状态。

**步骤：**

1. 登录 new API 管理后台。
2. 打开「设置 → 运营设置 / 通用」确认 **自用模式** 已关闭（`SelfUseModeEnabled = false`）。
3. 若需用 API 查：调用 `GET /api/misc/self_use_mode_enabled` 或查看 Option 中 `SelfUseModeEnabled` 为 false。

**验证：** 未配置倍率的模型在请求时应返回「倍率或价格未配置」类错误（或用户需勾选接受未设置倍率）。

---

### Task 2：选定一个用于验证的模型与通道

**目的：** 在自有 GCP service account 对应的通道上选一个模型，后续只在该模型上配倍率并跑请求。

**步骤：**

1. 在「通道」中找到类型为 **Vertex AI** 或 **Google Gemini**（或你们实际使用的 GCP 通道类型）的通道，确认其使用自有 GCP 项目/Service Account。
2. 查看该通道的 **模型列表**（channel 的 `models` 字段），记下一个要上架的模型名，例如 `gemini-2.0-flash` 或 Vertex 的 `gemini-1.5-flash`（以实际列表为准）。
3. 确认该通道状态为启用、有可用 Key。

**记录：** 通道 ID、通道类型、选定模型名（与请求时 `model` 参数完全一致）。

---

### Task 3：为选定模型添加倍率配置

**目的：** 在 new API 中为该模型配置 model_ratio（及可选 completion_ratio），使请求不再因「未配置倍率」被拒。

**方式 A — 管理后台：**

1. 进入「设置 → 倍率 / 模型定价」（或「模型倍率」相关页，如 `ModelRatioSettings.jsx` 对应页面）。
2. 找到 ModelRatio 的编辑入口（多为 JSON 或表格式），添加一条：`"<你选的模型名>": <数值>`，例如 `"gemini-2.0-flash": 0.15`（数值可先按 GCP 定价估算：输入约 $0.075/1M tokens 时可设约 0.15，即 1M tokens = 0.15 单位 quota，具体见项目 USD 常量）。
3. 保存（会调用 `PUT /api/option/`，key=ModelRatio）。

**方式 B — 直接调 API（需 Root/管理员鉴权）：**

1. `GET /api/option/` 拿到当前 options，取 `ModelRatio` 的 value（JSON 字符串）。
2. 解析 JSON，添加 `"<模型名>": <ratio>`，再 `PUT /api/option/` 传回 `key: "ModelRatio", value: "<新 JSON 字符串>"`。

**可选：** 若该模型有单独的输出倍率，在 CompletionRatio 中同样加一条；否则使用默认 1.0 即可。

**验证：** 刷新定价页或 `GET /api/pricing`，确认该模型的 `model_ratio` 已出现且值正确。

---

### Task 4：运行一次推理请求（Python 脚本）

**目的：** 用选定模型发一次 Chat Completion，触发 new API 计费并拿到 usage。

**文件：** 可在项目根或任意目录创建脚本，例如 `scripts/verify_model_ratio_request.py`。

**步骤：**

1. 准备 new API 的 base URL 和 API Key（或 Token）。
2. 使用 OpenAI 兼容接口：`POST /v1/chat/completions`，`model` = 你在 Task 2 选的模型名，body 中 `max_tokens` 设小一点（如 50）便于估算。
3. 脚本中打印响应里的 `usage`（prompt_tokens, completion_tokens）以及 HTTP 状态；若为 200，说明倍率已生效且未因「未配置倍率」被拒。

**示例（占位，按实际环境改）：**

```python
# scripts/verify_model_ratio_request.py
import os
import requests

BASE = os.environ.get("NEWAPI_BASE", "https://your-newapi-host/v1")
KEY = os.environ.get("NEWAPI_KEY", "sk-...")

r = requests.post(
    f"{BASE}/chat/completions",
    headers={"Authorization": f"Bearer {KEY}"},
    json={
        "model": "gemini-2.0-flash",  # 替换为 Task 2 选定模型名
        "messages": [{"role": "user", "content": "Say hello in one sentence."}],
        "max_tokens": 50,
    },
    timeout=60,
)
print("Status:", r.status_code)
if r.ok:
    data = r.json()
    print("Usage:", data.get("usage"))
else:
    print("Error:", r.text)
```

**验证：** 状态 200，且返回中有 usage；在 new API 的消费日志/用量记录中能看到该次请求。

---

### Task 5：核实 new API 收费（扣费与日志）

**目的：** 确认本次请求按配置的倍率正确扣除了用户 quota。

**步骤：**

1. 在 new API 管理后台查看 **消费日志 / 用量日志**（或调用相关日志 API），找到刚才那次请求：
   - 查看扣费额度（quota）或等值展示（USD/CNY/TOKENS，取决于运营设置）。
   - 对照公式：quota ≈ (prompt_tokens/1e6 × model_ratio + completion_tokens/1e6 × model_ratio × completion_ratio) × group_ratio × 单位系数（见 `common` 中 QuotaPerUnit 等）。
2. 若前端有「我的额度」或「消费记录」，确认余额或记录变化与预期一致。

**验证：** 扣费额与用 token 数、配置的 model_ratio（及 completion_ratio）相符。

---

### Task 6：核实 GCP 账单

**目的：** 在 GCP 侧确认该次推理产生了对应用量，便于后续做「new API 倍率 ↔ GCP 成本」的校准。

**步骤：**

1. 登录 **Google Cloud Console**，选择与 new API 该通道对应的项目（即提供 Service Account 的项目）。
2. 打开 **Billing → 报表 / 费用**，按服务筛选 **Vertex AI** 或 **Generative AI**（视实际计费产品）。
3. 按时间筛选到刚才请求的日期，查看该模型/API 的请求次数与 token 用量（或费用）；可与 new API 日志中的 token 数交叉验证。

**验证：** GCP 报表中能看到与本次测试时间、用量相符的计费或用量记录。

---

### Task 7：整理与后续

**目的：** 把「上架一个模型」的流程固化下来，便于后续批量上架。

**步骤：**

1. 在内部文档中记录：**上架流程 = 通道配置模型 → 在 ModelRatio（及必要时 CompletionRatio/ModelPrice）中配置该模型名 → 发请求验证 → 对 new API 扣费 + GCP 账单**。
2. 若需要多模型或从 OpenRouter 等同步倍率，可再使用「设置 → 倍率同步」或 `controller/ratio_sync.go` 的接口做增量同步，并做一次同样的 E2E 抽查。

---

## 三、注意事项与排错

- **报错「模型 xxx 倍率或价格未配置」**：说明该模型名在 `GetModelRatio`/`GetModelPrice` 中查不到。检查 (1) 倍率表里 key 是否与请求的 `model` 完全一致（含大小写、空格）；(2) Option 是否已保存并生效（可重启服务或等定时从 DB 同步）；(3) 是否误用了映射后的上游名（若 compact 后缀则 key 需带后缀）。
- **扣费为 0 或与预期不符**：检查分组倍率（group_ratio）、是否免费模型（model_ratio=0 或 model_price=0）、QuotaSetting 中是否关闭了免费模型预扣等。
- **GCP 无对应用量**：确认通道确实打到该 GCP 项目（看 channel 的 key/base_url 与项目）、请求成功（2xx）且未走缓存或其它通道。

---

## 四、执行选择

本方案为「摸清流程 + 单模型验证」的操作计划，可按上述 Task 1～7 在自有环境中逐项执行，无需按代码级 TDD 拆 commit。若你希望由 Agent 在仓库内完成具体代码改动（例如仅新增一个验证脚本或文档），可指定具体范围；否则按本文在管理后台与 GCP Console 中手动跑通即可。下周内在一个模型上完成 E2E 验证即达目标。

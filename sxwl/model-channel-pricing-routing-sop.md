# 模型-渠道-价格-路由配置 SOP（面向使用者设置与后端渠道配置联动）

本文用于统一以下三类配置，减少“看起来配置了但实际不可用/价格错误”的问题：

1. 渠道连通性配置：API endpoint、API credentials、API 调用约定一致性  
2. 渠道价格设置：输入、输出、缓存  
3. 用户模型联通性配置：模型名称（OpenRouter 风格）、模型到渠道路由关系、按售价路由策略（低价/高价）

---

## 0. 先理解数据链路（最关键）

在本项目中，展示与计费相关链路可抽象为：

- 渠道能力（channel + ability）  
  -> 模型可用性（model 可在哪些分组/端点中使用）  
  -> 价格参数（ModelPrice 或 ModelRatio/CompletionRatio/CacheRatio...）  
  -> 分组倍率（group_ratio + group-special-ratio）  
  -> 用户侧模型列表/价格展示与真实计费。

### 重要规则

- `quota_type = 0`：按量计费（token）  
- `quota_type = 1`：按次计费（request/task）  
- 按量计费下展示价（USD）核心关系：
  - `InputPrice(USD per 1M) = ModelRatio * 2 * GroupRatio`
  - `CompletionPrice = InputPrice * CompletionRatio`
  - `CacheReadPrice = InputPrice * CacheRatio`
  - `CacheCreatePrice = InputPrice * CreateCacheRatio`

---

## 1. 渠道连通性配置 SOP（先通，再谈价格）

### 1.1 渠道信息最小闭环

每个渠道至少确认：

- 渠道类型（OpenAI/OpenRouter/Gemini/...）
- Base URL
- Endpoint 约定（path + method）
  - 例如 OpenAI 风格：`POST /v1/chat/completions`
- 凭证（API Key/Secret 等）
- 模型名映射（上游真实名 vs 平台暴露名）

### 1.2 连通性检查步骤

1. 新建/编辑渠道，填写 Base URL 与 Key  
2. 用“渠道测试”功能做最小请求（一个最小 messages）  
3. 分别测试：
   - 非流式（stream=false）
   - 流式（stream=true）
4. 如果失败，优先排查：
   - endpoint path 是否和渠道协议一致
   - key 是否为空/被禁用
   - 模型名是否是该上游实际支持名称
   - 某些渠道需要特定 header（如 Bearer）

### 1.3 一致性要求（避免“能测通但线上异常”）

- 同一模型在不同渠道的调用格式应一致（至少 request schema 一致）。  
- 若供应商不支持某 endpoint，不要强行复用 OpenAI endpoint。  
- 使用模型元数据中的 endpoint 能力信息时，必须保证 path/method 可实际执行。

---

## 2. 价格配置 SOP（输入/输出/缓存）

### 2.1 配置优先级

按模型配置时只选一种主模式：

- 模式 A：按次价格（`ModelPrice`）  
- 模式 B：按量倍率（`ModelRatio` + 各项 ratio）

不要长期混用；混用只用于迁移窗口。

### 2.2 推荐配置顺序（按量计费）

1. 先配置输入价（或输入倍率）  
2. 再配置补全倍率/价格  
3. 最后配置缓存读取、缓存创建、图片/音频扩展倍率

推荐换算：

- `ModelRatio = InputPrice / 2`
- `CompletionRatio = OutputPrice / InputPrice`
- `CacheRatio = CacheReadPrice / InputPrice`
- `CreateCacheRatio = CacheCreatePrice / InputPrice`

### 2.3 分组价核对

完成后在“分组价格”中逐项核对：

- Group A/B/C 的输入价是否按倍率变化
- auto 组调用链路是否符合预期
- 计费类型（按量/按次）显示是否正确

---

## 3. 用户模型联通性与路由 SOP

### 3.1 模型命名规范（OpenRouter 风格）

- 对外暴露统一模型名（建议 OpenRouter 风格）  
- 在内部维护“模型名 -> 渠道真实名”映射  
- 新增渠道前先确认该模型是否同名可直通；不可直通则显式映射

### 3.2 模型到渠道绑定

每个用户可见模型至少绑定 1 个健康渠道，建议 2 个以上冗余渠道：

- 主渠道：成本或质量优先
- 备渠道：故障/限流兜底

### 3.3 路由策略（按售价）

可配置两类策略：

- 低价优先（成本优先）：选择有效成本最低渠道
- 高价优先（质量优先）：选择有效成本更高或高优先级渠道

“有效成本”建议计算口径：

- `有效输入价 = 模型输入价 * 分组倍率（含 group special）`
- 再结合稳定性权重（成功率/延迟/超时）做二次排序

---

## 4. `$75 / 1M` 这类异常的标准排查

当某模型显示为异常高价（典型如 `$75`）时按此顺序排查：

1. 检查该模型是否显式配置了 `ModelPrice` 或 `ModelRatio`  
2. 若未显式配置，后端可能使用默认兜底倍率（例如 37.5）  
3. 检查前端是否把“兜底值”当作“已配置值”展示  
4. 查看 `group_ratio` 是否异常放大（>1）  
5. 检查是否存在 group special ratio 覆盖

当前仓库已修复展示误导：

- 未显式配置价格时，前端显示“未设置价格”而不是显示由默认兜底值推导出的金额。

---

## 5. 上线前验收清单（Checklist）

- [ ] 每个目标模型至少有 1 个可用渠道连通  
- [ ] endpoint path/method 与渠道协议一致  
- [ ] API key 可用且未过期  
- [ ] 模型命名映射已验证（含别名）  
- [ ] 按量模型输入/补全/缓存价格核对通过  
- [ ] auto 分组链路符合预期  
- [ ] 低价/高价路由策略在灰度用户下验证通过  
- [ ] 消费日志中的 `model_ratio/group_ratio/model_price` 与展示一致  
- [ ] 未配置定价模型不会展示伪价格（如 `$75` 兜底值）

---

## 6. 团队协作建议

- 把“连通性变更”和“定价变更”拆成独立变更单，避免一次改太多。  
- 新模型先在测试分组开通，完成 1 天观察后再全量。  
- 定价配置建议由单一入口维护（避免多人直接改 JSON 造成覆盖冲突）。


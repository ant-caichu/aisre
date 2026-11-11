# 运维故障应急 Agent 技术架构

> 范围：聚焦“知识推理层”，即故障发生后的**实体锚定、上下文组装、根因假设、恢复计划建议与最终操作指引生成**。  
> 架构上明确区分：**融合与上下文层（确定性） → 推理与生成层（LLM） → 校验与展示层（确定性）**。  
> 模型侧全部采用本地/私有化开源模型，不依赖公有云 API；按“小模型 PoC → 32B 主力 → 235B 只处理难题”分级，不部署 671B 级超大 MoE 作为起步。  
> 不展开：自动执行、审批、回滚、Agent 学习。执行与学习作为后续演进，不在本期架构主线内。

---

## 1. 目标与边界

### 1.1 目标

让 Agent 在故障发生时，能够：

1. 把告警/现象锚定到 UModel 中的真实实例；
2. 拉取实例级拓扑、变更、日志、指标、链路；
3. 结合 RAGFlow 中的 Runbook、历史工单、架构文档；
4. 生成**有证据、有置信度、可解释**的根因假设；
5. 生成**结构化、可审查**的恢复计划，并渲染为最终操作指引。

### 1.2 边界

本期只做“只读推理”：

- 不直接执行恢复动作；
- 不替人做高风险决策；
- 不把 LLM 生成的自由文本当成可执行命令；
- 不把 RAGFlow 的知识图谱当成 CMDB 拓扑。

---

## 2. 架构原则

1. **UModel 管“世界是什么”**：对象类型、实例、关系、变更、遥测关联。
2. **RAGFlow 管“过去怎么做”**：文档、Runbook、历史案例、变更说明。
3. **概念—实例必须显式对齐**：文档里的概念要能落到 UModel 实例，不能靠模型猜。
4. **固定 SQL，不让 LLM 生成 SQL**：LLM 只选查询模板和参数，查询走 CMDB 只读从库。
5. **证据优先**：每个结论都要能追溯到实体、时间、来源。
6. **默认只读，降级可用**：CMDB/RAGFlow 不可用时，返回带新鲜度标记的降级结果。
7. **知识、状态、证据分层**：不要把文档、拓扑、遥测混成一个大上下文。
8. **融合与生成分离**：融合层做确定性的检索、裁剪、证据化；生成层做概率性的假设与计划编排；两者不能混成一个“大 prompt”。
9. **生成输入四类齐全**：静态知识、动态拓扑、运行证据、变更时间线；只有静态+动态，容易生成通用排查手册。
10. **模型分级部署**：8B 做抽取/摘要，32B 做主力诊断和计划，235B 只处理低置信度、复杂多跳或重大故障；不部署 671B 级模型作为起步。

---

## 3. 总体架构

```text
┌──────────────────────────────────────────────────────────────────────┐
│ 体验与展示层                                                         │
│ 故障应急 Copilot / ChatOps / API / 工单系统嵌入                       │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────────┐
│ 校验与展示层（确定性）                                               │
│ Schema 校验 │ 证据引用检查 │ 风险/前置条件检查 │ 禁用动作拦截 │ 渲染  │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────────┐
│ 推理与生成层（LLM）                                                  │
│ Orchestrator │ Diagnosis Reasoner │ Plan Composer │ LLM Gateway      │
│ 模型池：8B 快速 / 32B 主力 / 235B 难题兜底（可选）                  │
│ Prompt/版本管理 │ 结构化输出 Schema │ 工具调用/预算控制                │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────────┐
│ 融合与上下文层（确定性）                                             │
│ Context Builder │ Entity Resolver │ Evidence Builder │ Cache         │
│ 概念—实例映射 │ 关系来源标注 │ 上下文裁剪 │ 置信度/新鲜度              │
└───────────────┬───────────────────────────────┬──────────────────────┘
                │                               │
┌───────────────▼──────────────┐  ┌─────────────▼──────────────────────┐
│ UModel 运维世界模型           │  │ RAGFlow 经验知识层                  │
│ EntitySet / EntitySetLink    │  │ 文档 / Runbook / 历史工单           │
│ 实例 / 关系 / 变更 / 遥测关联 │  │ KnowledgeGraph（可选）              │
│ 动态实体（CMDB 从库 SQL）     │  │ 混合检索 / 元数据过滤 / 引用        │
└───────────────┬──────────────┘  └─────────────┬──────────────────────┘
                │                               │
┌───────────────▼───────────────────────────────▼──────────────────────┐
│ 数据源层                                                             │
│ CMDB 主库/只读从库 │ 变更/发布系统 │ Metrics/Logs/Traces │ 文档/工单  │
└──────────────────────────────────────────────────────────────────────┘

支撑层：IAM / 密钥管理 / 审计 / 限流 / 监控 / 评测 / 配置中心
```

---

## 4. 分层说明

### 4.1 数据源层

| 数据源 | 内容 | 接入方式 |
| --- | --- | --- |
| CMDB 主库 | CI、关系、属性 | 不直接查；CDC/同步/管理接口 |
| CMDB 只读从库 | 实例、关系、属性 | 固定 SQL 模板 + 只读账号 |
| 变更/发布系统 | 发布、配置、扩缩容、证书、网络变更 | 事件流/API/CDC |
| 可观测数据 | Metrics、Logs、Traces | SLS/Prometheus/APM |
| 文档与工单 | Runbook、SOP、历史故障、架构文档 | RAGFlow 数据接入 |
| 事件/告警 | 告警、事件、SLO 异常 | 告警平台/事件总线 |

关键点：

- CMDB **从库查询**用于拓扑和实例，不压主库；
- 变更事件不要只靠从库，走 CDC/事件流，避免复制延迟漏掉“刚发生的变更”；
- 所有数据源都要带 `source`、`timestamp`、`freshness`。

### 4.2 知识与模型层

#### 4.2.1 UModel：运维世界模型

UModel 负责：

- `EntitySet`：服务、Pod、主机、MySQL 实例、Redis 实例、SLB、K8s 集群等；
- `EntitySetLink`：调用、部署、包含、托管、主从、同可用区等；
- `MetricSet / LogSet / TraceSet / EventSet`：遥测数据集；
- `DataLink`：实体与遥测的关联；
- `EntitySet.dynamic = true`：实例从 Storage 实时获取；
- `EntitySetLink.dynamic = true`：关系从 Storage 实时获取。

CMDB 对接方式：

```text
CMDB 只读从库
  → 稳定视图（v_service / v_pod / v_service_pod_rel / v_change）
  → UModel MySQL Storage（若版本支持）
  → EntitySet / EntitySetLink（dynamic=true）
```

如果所用 UModel 版本不支持 MySQL Storage：

- 方案 A：CMDB → SLS EntityStore 同步，UModel 查 EntityStore；
- 方案 B：实现自定义 Storage Provider/适配层；
- 方案 C：在应用层做联邦查询，UModel 只负责语义和关系，不直接查 CMDB。

#### 4.2.2 RAGFlow：经验知识层

RAGFlow 负责：

- Runbook、SOP、故障复盘、历史工单；
- 架构文档、变更说明、厂商文档；
- 可选的 KnowledgeGraph：增强多跳文档问答；
- 元数据过滤：`domain`、`service`、`entity_type`、`env`、`doc_type`、`version`。

RAGFlow **不负责**：

- 当前有哪些实例；
- 实例之间的运行时关系；
- 实例的实时状态；
- 拓扑影响面分析。

建议把 RAGFlow 的实体类型尽量对齐 UModel 的 `EntitySet` 名称，减少概念映射成本。RAGFlow 的 Embedding/Rerank 接本地 TEI/Infinity 服务，不调用公有云。

#### 4.2.3 概念—实例映射

这是融合层的核心能力，负责把“文档概念”落到“真实实例”；它不是 UModel 或 RAGFlow 的内置功能，而是独立的一套映射与解析服务。

核心能力：

1. **类型映射**：RAGFlow 概念类型 → UModel EntitySet；
2. **实例解析**：文档中的名称/别名/IP/环境 → UModel 实体 ID；
3. **关系来源标注**：`document_asserted` vs `runtime_observed`；
4. **证据关联**：文档 chunk、日志、指标、链路 → 实体 + 时间；
5. **置信度与人工校准**：低置信度不硬绑定。

建议的元数据表：

| 表 | 用途 |
| --- | --- |
| `concept_type_mapping` | 概念类型 → EntitySet |
| `instance_alias` | 别名/名称/IP → entity_id |
| `doc_entity_link` | 文档 chunk → entity_id |
| `relation_provenance` | 关系的来源、置信度、有效期 |

### 4.3 融合与上下文层（确定性）

这一层不生成结论，只负责把“静态知识 + 动态拓扑 + 运行证据 + 变更时间线”整理成结构化、可引用的 `IncidentContext`。它决定了 LLM 能看到什么、看不到什么。

#### Context Builder 的职责

输入：告警/事件 + 时间窗 + 症状。

输出：一个紧凑的、结构化的 `IncidentContext`：

```json
{
  "incident": {
    "id": "INC-20260501-001",
    "symptom": "checkout-service P99 latency high",
    "start_time": "2026-05-01T10:00:00Z",
    "environment": "prod"
  },
  "entities": [
    {"entity_id": "service:checkout", "type": "Service", "role": "symptom"},
    {"entity_id": "pod:checkout-7f9c", "type": "Pod", "role": "impacted"},
    {"entity_id": "mysql:order-db", "type": "MySQLInstance", "role": "suspect"}
  ],
  "relations": [
    {"src": "service:checkout", "type": "calls", "dst": "mysql:order-db"}
  ],
  "changes": [
    {"entity_id": "service:checkout", "type": "deploy", "time": "2026-05-01T09:50:00Z"}
  ],
  "evidence": [],
  "freshness": {"cmdb_lag_seconds": 3, "telemetry_delay_seconds": 15}
}
```

上下文构建原则：

- 只取相关子图，限制跳数和实体数；
- 先实例、后概念、再文档；
- 每条证据带 `entity_id`、`timestamp`、`source`；
- 给 LLM 的是“证据表”，不是原始图数据库 dump。

融合层的四类输入：

| 输入 | 来源 | 作用 |
| --- | --- | --- |
| 静态知识 | RAGFlow | Runbook、历史案例、架构文档 |
| 动态拓扑 | UModel + CMDB | 当前实例、关系、依赖、影响面 |
| 运行证据 | Metrics/Logs/Traces | 异常是什么、发生在哪个实例 |
| 变更时间线 | CDC/变更系统 | 最近改了什么、是否相关 |

只有“静态 + 动态拓扑”，LLM 容易生成通用排查手册；加上运行证据和变更，才能生成针对当前故障的操作指引。

### 4.4 推理与生成层（LLM）

这一层就是“把 RAGFlow 静态内容和 UModel 动态内容交给 LLM，生成最终操作指引”的层。但它不是一次简单的“拼接 + 生成”，而是受控的推理与编排。

#### 组件

| 组件 | 职责 |
| --- | --- |
| Orchestrator | 规划查询步骤、调用工具、控制预算；决定是否需要补充证据 |
| Diagnosis Reasoner | 生成根因假设、排序、补充证据、区分症状与根因 |
| Plan Composer | 从 Runbook/动作模板选择候选，填入实例参数，生成结构化计划 |
| LLM Gateway | 按 8B/32B/235B 分级路由、限流、降级、成本、审计、结构化输出约束 |
| Prompt/版本管理 | 管理诊断、计划、校验不同阶段的提示词和版本 |

> `Entity Resolver` 属于融合层，不属于生成层；`Validator` 放在下一层“校验与展示层”，避免生成层既当运动员又当裁判。

#### 工具清单

LLM 只能通过白名单工具访问数据：

```text
resolve_entity(mention, type_hint, env)
get_entity(entity_id)
list_related(entity_id, relation_types, depth)
list_instances(entity_type, filters)
get_recent_changes(entity_id, since)
get_telemetry(entity_id, signal, window)
search_docs(query, filters)
get_runbook(concept_type, symptom)
get_similar_incidents(entity_type, symptom)
```

每个工具对应固定查询模板，不接受自由 SQL。

#### 输出结构

```json
{
  "hypotheses": [
    {
      "cause": "新版本连接池配置导致连接耗尽",
      "confidence": 0.78,
      "evidence_ids": ["ev-1", "ev-2", "ev-3"],
      "missing": ["确认 max_connections 配置"]
    }
  ],
  "plan": [
    {
      "action_id": "rollback_deploy",
      "target": "service:checkout",
      "preconditions": ["确认版本差异"],
      "expected_signals": ["db_connections 下降", "P99 回落"],
      "risk": "medium",
      "runbook_ref": "rb-123"
    }
  ],
  "citations": ["umodel:service:checkout", "ragflow:doc-123#chunk-4"]
}
```

> 生成层输出的是结构化 `DiagnosisResult`；最终的自然语言操作指引由校验与展示层渲染，避免生成层直接对外输出未经校验的内容。

#### 为什么融合与生成要分开

- 融合层是确定性的：同样的输入应该得到同样的上下文，便于调试和评测；
- 生成层是概率性的：需要 LLM 做假设、取舍和语言组织；
- 如果混在一起，无法判断是“检索错了”还是“模型推理错了”；
- 生成层只消费结构化证据和候选 Runbook，不直接消费原始图/原始文档全集；
- 生成层可以按需调用工具补证据，但必须有预算和超时。

### 4.5 校验与展示层（确定性）

| 组件 | 职责 |
| --- | --- |
| Template Validator | 校验输出是否符合操作指引模板；不通过必须重新输出，不得进入后续校验 |
| Schema Validator | 校验 `Hypothesis`/`Plan`/`Guidance` 的 JSON Schema |
| Evidence Checker | 检查每个结论是否有证据引用、引用是否存在 |
| Risk Checker | 检查前置条件、风险等级、是否包含禁用动作 |
| Confidence Gate | 低于阈值时降级为“建议人工确认”，不输出确定性结论 |
| Repair Coordinator | 将校验失败原因结构化反馈给生成层，控制修复次数与降级 |
| Renderer | 把结构化结果渲染成人可读的操作指引、证据列表和待确认项 |

这一层不生成新结论，只做“验收”和“呈现”。最终输出是**操作指引**，不是可执行命令。

#### 4.5.1 模板校验：第一道硬闸门

原则：**大模型输出的操作指引必须先通过模板校验；不满足模板要求，必须重新输出，不能进入证据、风险等后续校验，也不能带病渲染。**

模板至少要求：

- 结论摘要；
- 置信度；
- 根因假设（至少 1 个，含 `evidence_ids`、`missing`）；
- 操作步骤（`step`、`action_id`、`target`、`reason`、`preconditions`、`expected_signal`、`risk`、`stop_condition`）；
- 待确认项；
- 引用列表；
- 是否需要人工复核。

模板应有两个合法分支：

1. **正常操作指引**：有足够证据，给出步骤；
2. **信息不足/无法安全建议**：允许不输出操作步骤，但必须说明缺什么、建议人工确认。

> 如果模板只允许“必须给步骤”，会逼模型在证据不足时编造。模板必须允许 abstain。

校验规则：

- 模板要版本化，模板变更需要回归测试；
- 模板不通过 → 把缺失/不合规字段反馈给 LLM → **必须重新输出**；
- 重新输出时只修复模板结构，不得新增无证据事实，不得改变目标实例；
- 连续 2 次模板不通过 → 终止生成，返回“输出不合规”，不渲染操作指引；
- 模板校验通过后，才进入 Schema、证据、前置条件、风险、计划一致性、置信度校验。

#### 4.5.2 校验失败反馈与定向修复

原则：**校验失败后不允许“原样重试”或“重试到过为止”。必须先把失败原因结构化反馈给生成层，让 LLM 结合失败原因做定向修复；修复次数上限 1–2 次，仍失败则降级或转人工。**

其中，**模板校验失败是硬失败**：不进入证据、风险等后续校验，也不允许 warn/降级，必须把模板缺失项反馈给 LLM 并要求重新输出；连续 2 次模板失败则终止生成。

反馈给生成层的内容至少包括：

- 失败的 gate/check（模板、Schema、证据、前置条件、风险、计划一致性、置信度）；
- 模板失败时：缺失的章节/字段、格式不合规项、必须重新输出的指令；
- 失败的步骤、字段、证据 ID 或动作 ID；
- 失败原因（例如“缺少 stop_condition”“action_id 不在动作目录”“步骤 3 缺少 evidence_ids”“前置条件 deploy.exists=false”）；
- 修复约束（只修复失败项、不得新增动作目录外的动作、必须补证据、不得改变目标实例）；
- 可选的候选动作、候选证据或需要补充查询的提示。

反馈结构示例：

```json
{
  "attempt": 1,
  "status": "fail",
  "failed_checks": [
    {
      "gate": "action_catalog",
      "step": 2,
      "reason": "action_id 'restart_all' 不在动作目录",
      "allowed_actions": ["restart_pod", "rollback_deploy"]
    },
    {
      "gate": "evidence",
      "step": 3,
      "reason": "缺少 evidence_ids",
      "required": "至少引用一条指标或日志证据"
    }
  ],
  "repair_instructions": [
    "只修复失败步骤，不要重写整个计划",
    "不得新增动作目录外的动作",
    "保持原 JSON Schema 和目标实例不变"
  ]
}
```

修复流程：

```text
生成 → 模板校验
  ├─ 模板不通过：反馈缺失字段，必须重新输出；不进入后续校验
  │    └─ 连续 2 次不通过 → 终止生成，返回“输出不合规”
  └─ 模板通过 → 后续校验
       ├─ pass：渲染输出
       ├─ warn：带“待确认/数据可能过期”渲染
       └─ fail：按失败类型处理
            ├─ Schema 失败：带 Schema 错误定向重写
            ├─ 证据缺失：先补检索，再定向修复；仍缺则降级
            ├─ 前置条件为 false：移除动作或换安全方案
            ├─ 前置条件 unknown：标记人工确认，不默认通过
            ├─ 高风险/越权动作：直接阻断，不重试
            └─ 低置信度：输出鉴别诊断和缺失信息
     ↓
  定向修复（最多 1–2 次）
     ↓
  仍失败 → 降级/转人工
```

修复约束：

- 只反馈失败项，不重放全量上下文；
- **模板失败必须重新输出，不能进入后续校验，也不能直接降级/渲染**；
- 修复提示必须带“不得新增动作、不得越权、不得编造证据”等硬约束；
- 安全类失败（越权、跨租户、禁用动作、数据删除）**不重试，直接阻断或转人工**；
- 每次修复记录 `attempt`、耗时、token、失败原因，进入评测集；
- 两次修复仍失败，不再重试，进入降级阶梯。

### 4.6 体验层

- 故障应急 Copilot：输入告警/工单，输出根因假设、证据、恢复建议；
- ChatOps：在钉钉/飞书/Slack 中查询和确认；
- API：供工单、告警、自动化平台调用；
- 所有输出都带证据链接和置信度，不直接给“命令”。

### 4.7 支撑层

- IAM/RBAC：按环境、业务线、租户隔离；
- 密钥管理：CMDB 只读账号、本地模型网关 Token；
- 审计：谁在什么时间查了什么、生成了什么建议；
- 限流/配额：按用户、事件、工具；
- 监控：查询延迟、错误率、token 成本、检索命中率；
- 评测：离线集、回归测试、人工评分。

### 4.8 模型选型与本地部署策略

本方案不使用公有云模型 API，模型全部本地/私有化部署。模型策略不是“越大越好”，而是按任务分级：

> **小模型 PoC → 32B 主力 → 235B 只处理难题；不部署 671B 级超大 MoE 作为起步。**

#### 4.8.1 模型分层

| 层级 | 模型 | 用途 | 部署建议 |
| --- | --- | --- | --- |
| 轻量 | Qwen3-8B / Qwen3-4B | 实体抽取、摘要、查询改写、意图分类、工具参数归一 | 1×L40S 48GB，可多副本 |
| 主力 | Qwen3-32B | 默认诊断、假设排序、计划编排、工具调用 | 2×H100 80GB（FP8）；BF16 建议 4×H100 |
| 难题 | Qwen3-235B-A22B | 32B 低置信度、复杂多跳/因果、P0/P1 兜底 | 8×H100 80GB，按需/异步，可选 |
| Embedding | BGE-M3（或 Qwen3-Embedding-4B） | 文档、别名、相似故障召回 | TEI/Infinity，GPU/CPU |
| Rerank | BGE-reranker-v2-m3（或 Qwen3-Reranker-4B） | 召回结果精排 | TEI，GPU |
| OCR/解析 | PaddleOCR / MinerU | PDF、表格、扫描件 | CPU/GPU |
| VLM（可选） | Qwen2.5-VL-7B | 图表、截图、图片型 PDF | 1×L40S |

#### 4.8.2 路由策略

- **8B**：实体抽取、文档摘要、查询改写、意图分类、工具参数归一；不负责最终根因结论。
- **32B**：默认诊断与计划模型；大多数 P2/P3 和多数 P1 都走 32B。
- **235B**：只处理以下情况：
  - 32B 置信度低于阈值；
  - 32B 连续两次校验失败；
  - 复杂多跳依赖或因果推理；
  - P0/P1 重大故障；
  - 用户显式要求深度分析。
- 235B 走异步队列，设置并发、超时和 token 预算；不可用时回退 32B。
- **不部署 671B 级模型**；只有评测证明“瓶颈在模型能力而非检索/证据/Prompt”，且 GPU 预算获批后，才重新评估。

#### 4.8.3 部署阶段

| 阶段 | 模型组合 | GPU 建议 | 目标 |
| --- | --- | --- | --- |
| PoC | 8B + BGE-M3 + reranker + PaddleOCR | 1×L40S 48GB | 跑通链路，10–20 个历史故障 |
| MVP | 32B（FP8）+ 8B + embedding/rerank | 2×H100 80GB + 1×L40S | 30–50 个故障，对比 8B/32B |
| 生产 | 32B 多副本 + 8B + 可选 235B | 4–8×H100 80GB + L40S | 并发、HA、难题路由 |
| 高质量兜底 | 235B | 8×H100 80GB | 只服务难题，不承载全量流量 |

#### 4.8.4 服务栈

- **LLM 推理**：vLLM 或 SGLang，OpenAI-compatible API，支持 Tensor Parallel、FP8、guided JSON；
- **Embedding/Rerank**：TEI 或 Infinity；
- **模型网关**：LiteLLM 或自研，负责路由、限流、降级、成本统计；
- **编排**：Kubernetes + GPU Operator，模型权重放本地 NFS/S3，禁止运行时下载；
- **RAGFlow**：接本地 OpenAI-compatible endpoint，不访问公有云；
- **离线环境**：模型仓库、校验和、许可证清单、升级包全部内网管理。

#### 4.8.5 评测与升级门槛

- 用同一批 30–50 个历史故障对比 8B、32B、235B；
- 指标：根因 Top-1/Top-3、证据覆盖率、计划可用性、危险建议率、P95 延迟、GPU 成本；
- 只有当 32B 的失败主要来自“模型能力不足”，而不是检索缺失、证据不足或 Prompt 问题时，才引入 235B；
- 引入 235B 前，先尝试 32B + LoRA 微调、检索优化、工具契约优化；
- Embedding/Rerank 版本必须与索引绑定，换模型要重建索引。

#### 4.8.6 约束

- 许可证合规：Qwen、BGE、PaddleOCR、MinerU、Llama 等逐项确认商用条款；
- 数据不出内网，模型权重和推理日志都在私有环境；
- GPU 容量、功耗、机房资源要提前规划；
- 模型版本、Prompt 版本、索引版本必须可追溯、可回滚。

---

## 5. 核心流程

```text
1. 告警/事件进入
2. Entity Resolver 锚定实体（融合层）
3. UModel 查询实例子图 + 关系
4. CMDB 固定 SQL 查询实例/关系（只读从库）
5. 变更事件时间线合并
6. RAGFlow 检索 Runbook/历史案例
7. Evidence Builder 生成证据表 + 结构化 IncidentContext（融合层结束）
8. Diagnosis Reasoner 生成并排序根因假设（生成层）
9. Plan Composer 从 Runbook/动作模板生成结构化 RecoveryPlan（生成层）
10. 模板校验（不通过必须重新输出，连续 2 次不通过则终止）
11. Schema/Evidence/Risk/Confidence 校验（校验层）
12. 若 fail：把失败 gate、步骤、原因和修复约束反馈给生成层，定向修复（最多 1–2 次）
13. 若仍 fail：进入降级阶梯或转人工，不输出确定性操作指引
14. Renderer 渲染为最终操作指引：结论、证据、步骤、前置条件、风险、待确认项
15. 返回结果，并记录 trace（含模板重输出次数、修复轮次、失败原因）
```

四类输入必须齐备：静态知识、动态拓扑、运行证据、变更时间线。缺任何一类，生成层都应该降低置信度或明确标注“信息不足”。

每一步都要记录 trace：查询模板、参数、耗时、返回行数、文档命中、token 消耗、修复轮次与失败原因。

---

## 6. 关键数据契约

### 6.1 EntityRef

```json
{
  "entity_id": "service:checkout",
  "entity_type": "Service",
  "name": "checkout-service",
  "source": "umodel",
  "confidence": 0.95
}
```

### 6.2 Evidence

```json
{
  "evidence_id": "ev-1",
  "entity_id": "mysql:order-db",
  "type": "metric",
  "summary": "db_connections 达到上限",
  "timestamp": "2026-05-01T10:03:00Z",
  "source": "prometheus",
  "confidence": 0.9
}
```

### 6.3 Hypothesis

```json
{
  "hypothesis_id": "h-1",
  "cause": "连接池配置错误",
  "confidence": 0.78,
  "evidence_ids": ["ev-1", "ev-2"],
  "missing": ["max_connections 配置"]
}
```

### 6.4 Plan

```json
{
  "plan_id": "p-1",
  "action_id": "rollback_deploy",
  "target": "service:checkout",
  "preconditions": ["确认变更单"],
  "expected_signals": ["P99 下降"],
  "risk": "medium",
  "runbook_ref": "rb-123",
  "requires_approval": true
}
```

### 6.5 Guidance

```json
{
  "summary": "疑似新版本连接池配置导致连接耗尽",
  "confidence": 0.78,
  "evidence": ["ev-1", "ev-2", "ev-3"],
  "steps": [
    {
      "step": 1,
      "action": "核对新版本连接池配置",
      "target": "service:checkout",
      "expected": "确认 max_connections 是否被调小",
      "risk": "low"
    },
    {
      "step": 2,
      "action": "回滚到上一版本",
      "target": "service:checkout",
      "preconditions": ["确认变更单", "确认无其他兼容性依赖"],
      "expected": "连接数下降，P99 回落",
      "risk": "medium",
      "requires_approval": true
    }
  ],
  "citations": ["umodel:service:checkout", "ragflow:runbook-123#chunk-4"]
}
```

> `Guidance` 是给人看的最终操作指引；`Plan` 是机器可校验的结构化计划。本期不执行，但保留 `Guidance → Plan` 的映射关系，便于后续接执行层。

---

## 7. 部署架构

```text
生产数据区
  ├─ CMDB 主库
  ├─ CMDB 只读从库
  ├─ SLS / Prometheus / APM
  └─ 变更/发布系统

AI 知识区
  ├─ UModel 服务
  ├─ RAGFlow
  ├─ 概念—实例映射库（MySQL/PostgreSQL）
  ├─ Embedding/Rerank 服务（TEI/Infinity）
  ├─ OCR/文档解析服务（PaddleOCR/MinerU）
  ├─ 缓存（Redis）
  └─ 文档对象存储

模型推理区
  ├─ 8B 快速模型池（抽取/摘要）
  ├─ 32B 主力模型池（诊断/计划）
  ├─ 235B 难题模型池（可选，按需/异步）
  ├─ 本地模型仓库（NFS/S3）
  └─ vLLM/SGLang 推理服务

推理与生成区
  ├─ Orchestrator
  ├─ Diagnosis/Plan Services
  ├─ LLM Gateway（8B/32B/235B 路由）
  └─ Prompt/版本管理

校验区
  ├─ Schema/Evidence/Risk Validator
  └─ Confidence Gate

接入区
  ├─ API Gateway
  ├─ Copilot/Web
  └─ ChatOps

支撑区
  ├─ IAM / 密钥管理
  ├─ 审计 / 监控 / 日志
  └─ 评测平台
```

网络要求：

- CMDB 从库只允许 UModel/查询服务访问；
- RAGFlow 与文档源走内网；
- LLM/Embedding/Rerank 全部走内网模型网关，禁止公网直连和运行时下载权重；
- 所有服务使用统一 trace ID，便于串联。

---

## 8. 非功能设计

### 8.1 性能

- Context Builder 设定超时预算，例如 3–5 秒；
- 查询模板强制 `LIMIT`；
- 热点实体/拓扑缓存；
- 文档检索 top-k + rerank，控制上下文长度；
- 默认走 32B；8B 处理抽取/摘要/查询改写；235B 只按低置信度、复杂多跳、P0/P1 触发；
- 235B 走异步队列，设置并发、超时和 token 上限，不可用时回退 32B；
- 生成层单独设 token/工具调用预算，避免无限追问；
- 校验失败修复最多 1–2 次，每次只反馈失败项；安全类失败不重试；修复预算计入总 token/时延；
- 模板重输出最多 2 次，计入总时延/token；连续失败直接终止，不进入语义校验；
- 校验层以规则/Schema 为主，必须轻量、确定性，不能再次调用大模型做“自我评审”。

### 8.2 一致性

- CMDB 从库存在复制延迟，返回 `replication_lag`；
- 变更事件走 CDC，不依赖从库；
- 数据缺失标记 `incomplete`，不让 LLM 把“没查到”当成“没有”；
- 关键拓扑可回源主库，但要限流。

### 8.3 安全

- CMDB 只读账号，只授予视图 `SELECT`；
- 固定 SQL 模板 + 参数化；
- 禁止 LLM 直接访问 DB；
- 文档内容视为不可信输入，防止提示注入；
- 输出脱敏；
- 全链路审计。

### 8.4 降级

- CMDB 不可用：返回最近缓存拓扑，标记 stale；
- RAGFlow 不可用：只做拓扑诊断，不给 Runbook 引用；
- LLM 不可用：返回证据表和规则匹配结果；
- 任一工具超时：跳过并记录，不阻塞整体。

---

## 9. 评测体系

离线评测集：

- 30–50 个历史故障；
- 人工标注：涉及实体、根因、关键证据、正确处置；
- 回放 Agent 的实体锚定、上下文、假设排序、计划建议。

指标：

| 维度 | 指标 |
| --- | --- |
| 实体锚定 | Top-1/Top-3 准确率 |
| 上下文 | 关键证据召回率、无关证据比例 |
| 诊断 | 根因 Top-1/Top-3、证据引用准确率 |
| 计划 | 计划可用性、前置条件完整率、危险步骤率 |
| 生成 | 证据引用覆盖率、无依据结论率、Runbook 引用准确率 |
| 校验 | 危险建议拦截率、误拦截率、低置信度降级率 |
| 模板 | 模板一次通过率、平均重输出次数、模板失败终止率 |
| 修复 | 首次校验通过率、修复成功率、平均修复轮次、重复失败率 |
| 模型分层 | 8B/32B/235B 的根因/计划/延迟/成本对比 |
| 体验 | 生成时延、token 成本、专家评分 |

模型分层评测：用同一批 30–50 个故障对比 8B、32B、235B；只有确认 32B 的瓶颈在模型能力（而非检索、证据或 Prompt）时，才引入 235B，并同时评估 GPU 与延迟成本。

线上 shadow 模式：只记录建议，不执行，与人工处置对比。

---

## 10. MVP 路线

### Phase 1：8B PoC

- 1×L40S 48GB；
- Qwen3-8B + BGE-M3 + BGE-reranker-v2-m3 + PaddleOCR；
- 选 1–2 类对象：`Service` + `MySQLInstance`；
- CMDB 固定 SQL + 只读从库；
- UModel 建模实体/关系；
- RAGFlow 导入 Runbook/历史工单；
- 只做实体锚定、上下文、根因 Top-3、计划建议与最终操作指引；
- 10–20 个历史故障验证链路，不追求最终质量。

### Phase 2：32B 主力

- 2×H100 80GB 跑 Qwen3-32B FP8；
- 1×L40S 跑 Qwen3-8B、Embedding、Rerank；
- 建类型映射、别名表、文档—实体链接；
- 加入变更时间线；
- 扩展 Pod/Host/Redis/SLB；
- 加入证据表和置信度；
- 用 30–50 个历史故障对比 8B/32B，确认 32B 是否成为主力。

### Phase 3：235B 难题兜底

- 8×H100 80GB 部署 Qwen3-235B-A22B（可选）；
- 只对低置信度、复杂多跳、P0/P1 重大故障按需调用；
- 235B 走异步队列，不承载全量流量；
- 做在线 shadow 对比；
- 为后续执行/学习预留接口；
- **不部署 671B 级模型**，除非评测证明 32B/235B 仍存在明确的模型能力瓶颈且 GPU 预算获批。

---

## 11. 关键风险与决策点

| 风险 | 影响 | 应对 |
| --- | --- | --- |
| UModel 版本不支持 MySQL Storage | 动态实例查询无法按预期落地 | 确认版本；否则 SLS EntityStore/自定义 Provider |
| CMDB 关系粒度不足 | 查不到 Service→Pod | 建关系视图/中间实体/扁平化 |
| 从库复制延迟 | 漏掉最近变更 | 变更走 CDC；关键查询回源 |
| RAGFlow KG 被误当拓扑 | 关系不准 | 明确文档图 vs 运行时图 |
| LLM 幻觉/危险步骤 | 误导应急 | 固定动作模板 + 证据引用 + Validator |
| 模板不合规反复重写 | 生成循环、延迟上升 | 模板版本化；不通过必须重输出，最多 2 次；仍失败终止 |
| 校验失败反复重试 | 时延和 token 失控 | 只反馈失败项；最多 1–2 次；安全类失败不重试 |
| 文档提示注入 | 越权/错误输出 | 文档视为不可信数据；工具白名单 |
| 评测数据不足 | 无法证明有效 | 先积累 30–50 个历史案例 |
| GPU 容量/成本不足 | 32B/235B 无法稳定服务 | 8B 兜底；32B FP8；235B 只按需；量化与多副本 |
| 开源模型能力不足 | 根因/计划质量不达标 | 先优化检索/证据/Prompt；32B LoRA；必要时再上 235B |
| 许可证/供应链风险 | 商用合规或升级受阻 | 逐项核对许可证；离线模型仓库；权重校验 |

---

## 12. 一句话总结

> **UModel 提供实例级运维世界模型，CMDB 只读从库通过固定 SQL 提供动态实例与关系，RAGFlow 提供经验知识；融合层把四类输入整理成结构化证据上下文，推理与生成层让 LLM 做受控的诊断和计划编排，校验层保证每个结论有证据、可解释、不越权，最终输出给人看的操作指引。模型全部本地开源部署，按 8B → 32B → 235B 分级，不启用 671B 级模型作为起步。输出必须先通过模板校验，不满足模板必须重新输出；模板通过后再做证据、前置条件、风险等校验。校验失败时，先把失败原因反馈给生成层做定向修复，最多 1–2 次；仍失败则降级或转人工，不“重试到过为止”。**

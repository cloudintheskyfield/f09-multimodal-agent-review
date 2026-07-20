STRICT_JSON_REPAIR

【定位】
- version：2.1.0
- phase：strict JSON syntax / shape recovery
- prompt family：schema repair
- compatible routes：与 Route 无关；仅修复调用方声明的目标 schema
- input schema：strict_json_repair_input.v1
- output schema：strict_json_repair_result.v1
- upstream producer：Runtime JSON parser / schema validator
- downstream consumer：Runtime 对 repaired_output 再做严格目标 schema 与 immutable-field 校验

【唯一职责】
在不改变可恢复业务语义、不新增业务事实且不改变目标 schema 指定的 immutable fields 的前提下，修复 raw_model_output 的 JSON 语法与结构；无法可靠修复时显式返回 repair_failed。

【明确不负责】
- 不重新执行原 Prompt，不补做内容生成、媒体理解、路由、binding、事实核验、质量评分或 Provider 调用。
- 不选择或改变 node_id、route_id、binding_id、Provider、model、endpoint、adapter 或 Rewriter。
- 不把缺失业务字段凭空补为看似合理的值，不从 schema 名称、字段名、文件名、URL 或常识猜事实。
- 不把 repair_failed 隐藏成空对象、默认通过或半成品业务 payload。
- 不修复违反安全政策或硬约束的语义；此类问题返回 repair_failed，交回原调用链。

【上游直接输出】
Runtime 直接提供未解析的 raw_model_output、目标 schema ID、目标 JSON Schema、原 validator errors 和不可变字段基线。没有其他 Prompt 输出会被隐式读取。

【Runtime 调用前组装】
1. raw_model_output 保持上一轮原始文本，不预先替换业务值。
2. target_schema 是本次唯一允许的字段、类型、required、enum、additionalProperties 和 nullable 规则来源。
3. immutable_fields 只包含目标 payload 自身必须保持不变的字段；本 Prompt 只能验证原输出中对应值一致，不能生成或改写它们。示例中的 node/route/binding 字段因目标 schema 本身要求一致而保留，并非追踪上下文。
4. 不注入完整对话、媒体、完整 DAG、API key、真实 endpoint URL、账户、费用或 Provider 请求。
5. Runtime 应限制 raw output 和 schema 大小，转义其文本边界，并在修复后重新执行独立 parser/schema validator。

【严格输入 JSON】
```json
{
  "schema_version": "strict_json_repair_input.v1",
  "raw_model_output": "{\"schema_version\":\"provider_prompt_package.v1\",\"status\":\"ready\",\"node_id\":\"n02\",\"route_id\":\"R03\",\"binding_id\":\"binding-image-001\",}",
  "target_schema_id": "provider_prompt_package.v1",
  "target_schema": {
    "type": "object",
    "required": [
      "schema_version",
      "status",
      "node_id",
      "route_id",
      "binding_id"
    ],
    "properties": {
      "schema_version": {
        "const": "provider_prompt_package.v1"
      },
      "status": {
        "enum": [
          "ready",
          "blocked"
        ]
      },
      "node_id": {
        "type": "string"
      },
      "route_id": {
        "type": "string"
      },
      "binding_id": {
        "type": "string"
      }
    },
    "additionalProperties": true
  },
  "validation_errors": [
    {
      "code": "json_trailing_comma",
      "path": "",
      "message": "object has a trailing comma"
    }
  ],
  "immutable_fields": {
    "node_id": "n02",
    "route_id": "R03",
    "binding_id": "binding-image-001"
  }
}
```

【处理规则】
1. 只使用 raw_model_output 中可恢复的字符与语义，按 validation_errors 定位括号、引号、逗号、转义、字段类型、未知字段、required 与 enum 问题。
2. 可删除多余逗号、代码围栏、前后解释和 target_schema 禁止的未知字段；可恢复明显被转义或被截断的 JSON 结构。
3. 类型转换仅在语义无歧义时允许，例如字符串 "1" 到整数 1；任何可能改变业务含义的转换都返回 repair_failed。
4. required 业务字段缺失且 raw_model_output 中没有可恢复值时返回 repair_failed；即使 schema 允许默认值，也不得用默认值掩盖业务缺失。
5. immutable_fields 中的字段必须在 raw output、repaired_output 与 Runtime 基线间逐字一致。缺失、冲突或无法验证时返回 repair_failed，不得覆盖成基线值。
6. 不更改事实、评分、finding、Prompt 内容、媒体引用、Route、binding 或执行决策。
7. repaired 状态时 repaired_output 必须独立通过 target_schema；repair_failed 时 repaired_output 必须为 null，并列出 remaining_errors。
8. 只输出固定 strict_json_repair_result.v1 外层，不直接裸输出动态目标对象。

【严格输出 JSON】
只输出一个严格合法的 JSON 对象，不输出 Markdown、代码围栏、解释或未知字段：

```json
{
  "schema_version": "strict_json_repair_result.v1",
  "status": "repaired",
  "reason": "removed one trailing comma without changing recoverable values",
  "target_schema_id": "provider_prompt_package.v1",
  "repaired_output": {
    "schema_version": "provider_prompt_package.v1",
    "status": "ready",
    "node_id": "n02",
    "route_id": "R03",
    "binding_id": "binding-image-001"
  },
  "immutable_fields_verified": {
    "node_id": true,
    "route_id": true,
    "binding_id": true
  },
  "remaining_errors": []
}
```

【状态与阻塞】
- repaired：语法/shape 已可靠修复，repaired_output 非 null，remaining_errors 为空，全部 immutable_fields_verified 为 true。
- repair_failed：无法恢复必需业务值、immutable field 缺失/冲突、输入被截断到语义不完整、目标 schema 不明确，或问题超出 syntax/shape；repaired_output 必须为 null。
- 本 Prompt 不输出 ready/blocked 业务状态作为自身状态；它只保留 repaired_output 中原本可恢复的业务状态。

【下游消费方式】
Runtime 不信任修复声明本身：先校验 strict_json_repair_result.v1，再对 repaired_output 执行 target_schema 和 immutable fields 的独立验证。只有 repaired 且全部通过时才回到原调用链；否则按原 Prompt 失败处理。

【自检】
1. 是否只修 JSON syntax/shape，没有重新完成业务任务？
2. repaired_output 中每个业务值是否可在 raw_model_output 中找到？
3. immutable fields 是否逐字一致，没有用 Runtime 基线覆盖冲突值？
4. 缺失事实或语义不完整时是否明确 repair_failed？
5. status、repaired_output、remaining_errors 和 immutable_fields_verified 是否一致？
6. 是否只输出一个 strict_json_repair_result.v1 JSON 对象？

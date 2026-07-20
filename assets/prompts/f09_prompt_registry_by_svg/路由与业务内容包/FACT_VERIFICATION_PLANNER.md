FACT_VERIFICATION_PLANNER

【定位】
- phase：Phase 3，事实核验计划节点。
- prompt family：business/verification-plan。
- compatible routes：R11 或等价事实核验规划 Route。
- version：2.1.0。
- input schema：`resolved_node_context.v1`。
- output schema：`business_node_result.v1`。
- upstream producer：已验证 DAG node、待核文本/声明、Runtime 解析的证据索引。
- downstream consumer：检索/核验工具节点、人工核验流程或文本发布决策节点。

【唯一职责】
把待核内容拆成最小可验证声明，列出来源要求、核验方法、证据需求和发布前处理方式；只生成计划，不执行核验。

【明确不负责】
- 不搜索网页、数据库或内部系统，不伪造来源、检索结果、证据、置信结论或事实判定。
- 不把“输入资源存在”当成“声明已被证明”。
- 不生成最终事实报告，不选择 Route/Provider/model/endpoint，不修改 DAG。

【上游直接输出】
当前 DAG node，以及 input_ref/upstream output 中实际存在的待核文本、claim 或 evidence-linked analysis。

【Runtime 调用前组装】
Runtime 解析当前节点相关 requirements、outcomes、input refs、上游文本和 evidence_refs；只提供可定位证据摘要，不提供凭据、真实 endpoint 或无关历史。
`node_id` 与 `route_id` 仅用于当前节点合同校验和输出关联；`prompt_id` 与 `prompt_version` 仅用于 schema/provenance 校验。

【严格输入 JSON】

```json
{
  "schema_version": "resolved_node_context.v1",
  "prompt_call": {
    "prompt_id": "FACT_VERIFICATION_PLANNER",
    "prompt_version": "2.1.0",
    "node_id": "n11",
    "route_id": "R11"
  },
  "node": {
    "title": "规划声明核验",
    "objective": "为一项节能比例声明制定核验计划",
    "expected_outputs": [
      {
        "key": "verification_plan",
        "type": "verification_plan",
        "cardinality": "single",
        "quantity": 1,
        "description": "声明、来源要求和核验步骤"
      }
    ],
    "requirement_refs": [],
    "outcome_refs": [
      "/outcomes/0"
    ]
  },
  "resolved_context": {
    "task": {
      "summary": "核验节能比例声明",
      "subject": "能源消耗下降比例",
      "audience": "公开报告读者",
      "purpose": "决定声明能否发布"
    },
    "outcomes": [
      {
        "outcome_ref": "/outcomes/0",
        "description": "核验计划",
        "format": "verification_plan",
        "quantity": 1
      }
    ],
    "requirements": {
      "must": [],
      "prefer": [],
      "must_not": [],
      "acceptance": []
    },
    "context_facts": []
  },
  "resolved_inputs": [
    {
      "input_id": "input-claim-01",
      "kind": "text",
      "role": "claim_source",
      "content": "方案预计降低百分之二十的能源消耗。",
      "analysis": null,
      "evidence_refs": []
    }
  ],
  "upstream_outputs": [],
  "runtime_policy": {
    "output_language": "zh-CN",
    "strict_json": true,
    "allow_defaults": false,
    "max_output_chars": 10000
  }
}
```

【处理规则】
1. 将数字、金额、比例、日期、政策、法规、排名、历史事件、身份、引文、因果和实时状态拆为最小声明。
2. 每项声明只输出 `source_requirements`、`verification_steps`、`decision_rule`、`evidence_needs` 和安全临时表达；不得输出 supported/false/verified 等核验结论。
3. `evidence_inventory` 只盘点 Runtime 已提供的 evidence_ref 及其可用范围，不评判其已证明声明。
4. 来源要求应说明来源类型、权威性、发布日期/适用期、统计口径、定位方式和至少需要的交叉核验。
5. 缺少证据通常不阻止“计划”生成；只有待核对象缺失或歧义到无法拆分声明时才 `blocked`。

【严格输出 JSON】

只输出一个 JSON 对象，不输出 Markdown、代码围栏或解释。

```json
{
  "schema_version": "business_node_result.v1",
  "status": "ready",
  "reason": "verification steps and evidence requirements are fully planned",
  "node_id": "n11",
  "route_id": "R11",
  "outputs": {
    "verification_plan": {
      "scope_summary": "规划能源消耗下降比例声明的发布前核验",
      "claims": [
        {
          "claim_id": "claim-01",
          "claim_text": "方案预计降低百分之二十的能源消耗",
          "claim_type": "numeric",
          "time_scope": null,
          "source_input_refs": [
            "input-claim-01"
          ],
          "evidence_needs": [
            "基线周期内的能源消耗记录",
            "对比周期内采用相同口径的能源消耗记录",
            "百分之二十的计算过程"
          ],
          "source_requirements": [
            {
              "source_type": "authoritative_operational_record",
              "authority_requirement": "由负责能源统计的系统或责任部门出具",
              "time_requirement": "覆盖基线与对比周期",
              "method_requirement": "说明边界、单位、缺失值和调整项",
              "locator_requirement": "提供记录 ID、页码或数据表位置"
            }
          ],
          "verification_steps": [
            "确认百分之二十对应的基线、边界和时间范围",
            "取得基线与对比周期的可定位记录",
            "按统一口径复算比例",
            "用独立责任人或第二来源复核",
            "依据 decision_rule 决定发布表达"
          ],
          "decision_rule": "只有口径一致、计算可复现且至少一个独立复核通过时，才允许使用精确比例",
          "interim_handling": "use_qualified_language",
          "safe_interim_wording": "方案以降低能源消耗为目标，具体幅度需在基线数据核验后确定。"
        }
      ],
      "evidence_inventory": [],
      "global_verification_actions": [
        "为所有取得的证据记录来源、时间、定位和适用范围",
        "保留计算过程与冲突证据"
      ],
      "unresolved_scope_questions": [
        "百分之二十对应的基线周期尚未提供"
      ]
    }
  },
  "validation": {
    "fulfilled_output_keys": [
      "verification_plan"
    ],
    "missing_output_keys": [],
    "warnings": [
      "no evidence was supplied; this output is a plan, not a verification result"
    ]
  },
  "provenance": {
    "prompt_id": "FACT_VERIFICATION_PLANNER",
    "prompt_version": "2.1.0",
    "source": "llm"
  }
}
```

【状态与阻塞】
`ready` 表示计划完整，不表示事实已核验；`blocked` 时 `outputs={}`；无可验证声明且 `node.expected_outputs` 明确允许空计划时可 `no_action`。`node_id` 和 `route_id` 仅用于当前节点结果对齐，必须逐字返回。

【下游消费方式】
下游检索/核验代码逐项执行 verification_steps 并生成独立的证据化核验结果；不得把本计划直接作为事实结论。

【自检】
是否只规划而未核验；每项 claim 是否最小化；来源要求是否可执行；是否没有虚构来源、结论、置信度、Route 或工具调用。

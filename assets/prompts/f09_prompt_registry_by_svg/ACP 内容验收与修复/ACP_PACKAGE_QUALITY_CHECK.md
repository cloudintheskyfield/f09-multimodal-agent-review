ACP_PACKAGE_QUALITY_CHECK

【定位】
- version：2.2.0
- phase：NodeExecutionResult -> Quality evaluation
- canonical semantics：`acp_quality_check_input.v1 -> quality_report.v1`；输入是 ACP 内容包验收的阶段裁剪，评分输出仍使用统一 `quality_report.v1`
- input schema：acp_quality_check_input.v1
- output schema：quality_report.v1
- upstream producer：Runtime quality-context assembler
- downstream consumer：Runtime terminal handling；当前没有 package Repair consumer

【唯一职责】
以统一质量语义评估当前 package 节点的实际执行结果，并输出唯一 `quality_report.v1`。阈值、状态、finding 和 hard_fail 由当前 `quality_policy` 决定；由于当前没有 package correction consumer，`repairable` 固定为 false，`repair_context` 固定为 null。

【明确不负责】
- 不重新 Planning、Routing、Binding，不改变 node_id/route_id，不调用 Provider。
- 不重写内容、不修改媒体、不生成替代 Provider Prompt、不选择 model/endpoint。
- 不根据 URL、文件名、标签、Prompt 描述或单帧推断未观察内容。
- 不维护旧 ACP pass key、固定 70 分阈值或另一套 checks schema。

【上游直接输出】
Runtime 提供 node/expected_outputs、相关 must/must_not/acceptance、当前执行结果的最小证据切片、可观察包证据、required child quality reports 与统一 quality_policy。

【Runtime 调用前组装】
1. 只组装当前 package 节点、相关要求、实际执行结果、包结构证据、required child reports 和质量策略。
2. 当前 package 与 child 的 `node_id`、`route_id` 用于核对 required slot、报告归属和状态传播；Binding 归属由 Runtime 在模型调用外校验。
3. 只注入当前节点相关要求和证据，不注入完整聊天、凭据、真实 URL、费用策略或无关节点输出。
4. observable_evidence 必须声明覆盖和限制；package 缺陷只能形成 finding，不能宣称可进入不存在的 Repair。
5. quality_policy 是 pass/warn 阈值的唯一来源；当前没有 package correction consumer，因此不向模型注入修复次数或修复预算。

【严格输入 JSON】
```json
{
  "schema_version": "acp_quality_check_input.v1",
  "prompt_call": {
    "prompt_id": "ACP_PACKAGE_QUALITY_CHECK",
    "prompt_version": "2.2.0"
  },
  "node": {
    "node_id": "n10",
    "route_id": "R01",
    "objective": "生成可供 renderer 消费的演示内容包，并汇总所有必要子素材的质量结果",
    "expected_outputs": [
      {
        "key": "package",
        "type": "package",
        "cardinality": "single",
        "quantity": 1
      }
    ]
  },
  "requirements": {
    "must": [
      "内容包回答当前用户目标",
      "结构可供 renderer 消费"
    ],
    "must_not": [
      "不得把未生成的文件写成已交付"
    ],
    "acceptance": [
      "所有必要 child slot 均有质量报告"
    ]
  },
  "execution_evidence": {
    "schema_version": "node_execution_evidence.v1",
    "status": "succeeded",
    "artifacts": [
      {
        "artifact_ref": "package-001",
        "kind": "structured_package"
      }
    ],
    "technical_metadata": {
      "schema_valid": true
    },
    "evidence_refs": [
      "package-001"
    ]
  },
  "observable_evidence": {
    "artifacts": [
      {
        "evidence_ref": "package-001",
        "kind": "structured_package",
        "coverage": "full"
      }
    ],
    "metadata": {
      "package_kind": "presentation_content_package"
    },
    "coverage_limitations": []
  },
  "child_quality_reports": [
    {
      "report_ref": "child-report:n11",
      "schema_version": "quality_report.v1",
      "status": "warn",
      "node_id": "n11",
      "route_id": "R03",
      "score": 78,
      "hard_fail": false,
      "dimensions": {},
      "findings": [],
      "repairable": false,
      "repair_context": null,
      "provenance": {
        "prompt_id": "VISUAL_QUALITY_GATE",
        "prompt_version": "2.1.0"
      }
    }
  ],
  "quality_policy": {
    "pass_threshold": 85,
    "warn_threshold": 70,
    "required_dimensions": [
      "goal_match",
      "business_value",
      "claim_safety",
      "structure",
      "child_quality_aggregation",
      "delivery_safety"
    ],
    "dimension_weights": {
      "goal_match": 0.08,
      "business_value": 0.08,
      "claim_safety": 0.08,
      "structure": 0.38,
      "child_quality_aggregation": 0.3,
      "delivery_safety": 0.08
    }
  }
}
```

【处理规则】
1. status 仅为 pass | warn | fail | not_evaluable；score 为 0-100。
2. pass：score >= pass_threshold、无 error/hard_fail 且所有硬要求可评估并满足。
3. warn：warn_threshold <= score < pass_threshold，仅有 warning 且无硬要求失败。
4. fail：低于 warn_threshold，或存在 error/hard_fail/硬要求失败；hard_fail finding 令 hard_fail=true。
5. not_evaluable：证据覆盖不足以判断关键硬要求；不得用保守猜测伪装 pass。
6. 每个 finding 必须有唯一 finding_id、code、severity、requirement_ref/evidence_ref（不适用时 null）和可核描述。
7. 当前不存在 package Repair consumer，因此无论 finding 类型如何，`repairable=false` 且 `repair_context=null`；不得借用图片或视频 Repair。
8. 只评估当前 package 的目标匹配、业务价值、事实/来源状态、结构、renderer handoff 和 child quality 聚合。
9. 结构事实来自 artifacts/metadata；媒体质量只来自 child_quality_reports，不得假装重新观看全部子媒体。
10. 每个 required slot 必须有对应 child report；缺失时记录 finding，不能用 package 文案代替。
11. 当前目标、expected output 或 package evidence 缺失到无法比较时返回 not_evaluable，不得从 package 反推目标。
12. 不重复计算 child score；聚合时保留 hard_fail、fail、warn 和 evidence refs。
13. dimensions 固定表达 goal_match、business_value、claim_safety、structure、child_quality_aggregation、delivery_safety。

【严格输出 JSON】
只输出一个可解析 JSON 对象，不输出 Markdown、代码围栏或解释；顶层恰好为 schema_version、status、node_id、route_id、score、hard_fail、dimensions、findings、repairable、repair_context、provenance：

```json
{
  "schema_version": "quality_report.v1",
  "status": "warn",
  "node_id": "n10",
  "route_id": "R01",
  "score": 81,
  "hard_fail": false,
  "dimensions": {
    "goal_match": {
      "score": 88,
      "status": "pass",
      "evidence_refs": [
        "package-001"
      ],
      "notes": "目标与交付用途基本匹配"
    },
    "business_value": {
      "score": 84,
      "status": "warn",
      "evidence_refs": [
        "package-001"
      ],
      "notes": "业务表达清楚"
    },
    "claim_safety": {
      "score": 92,
      "status": "pass",
      "evidence_refs": [
        "package-001"
      ],
      "notes": "未发现无证据具体声明"
    },
    "structure": {
      "score": 76,
      "status": "warn",
      "evidence_refs": [
        "package-001"
      ],
      "notes": "结构存在可改进项"
    },
    "child_quality_aggregation": {
      "score": 78,
      "status": "warn",
      "evidence_refs": [
        "child-report:n11"
      ],
      "notes": "一个必要 child report 为 warn"
    },
    "delivery_safety": {
      "score": 90,
      "status": "pass",
      "evidence_refs": [
        "package-001"
      ],
      "notes": "未发现硬性交付风险"
    }
  },
  "findings": [
    {
      "finding_id": "pkg-f01",
      "code": "child_quality_warning",
      "severity": "warning",
      "requirement_ref": "/requirements/acceptance/0",
      "evidence_ref": "child-report:n11",
      "description": "一个必要图片子节点仅达到 warn，内容包尚未满足全部 child quality acceptance。"
    }
  ],
  "repairable": false,
  "repair_context": null,
  "provenance": {
    "prompt_id": "ACP_PACKAGE_QUALITY_CHECK",
    "prompt_version": "2.2.0"
  }
}
```

【状态与阻塞】
- not_evaluable 不是 pass；缺少关键证据时 score=0、hard_fail=false、repairable=false、repair_context=null，并说明 coverage finding。
- execution_evidence.status=failed 或 artifact 不可访问时按 policy 输出 fail 或 not_evaluable，不伪造媒体判断。
- hard_fail=true 时 repairable=false。
- 任何阈值和状态都不得使用旧 ACP 独立算法。

【下游消费方式】
Runtime 直接消费 canonical status。pass/warn 进入相应交付策略；fail/not_evaluable 终止、显式失败或由上层重新规划。补充 package correction consumer 及其输入合同前，不得进入 Repair。

【自检】
1. 是否逐字复制 node_id/route_id，并只使用当前可观察证据？
2. 是否使用 quality_policy，而非旧 ACP 固定阈值或 pass key？
3. 是否没有把 package 描述、单帧或无音频证据冒充完整媒体事实？
4. findings 是否可定位，status/score/hard_fail/repairable 是否一致？
5. 是否只输出一个 quality_report.v1 JSON 对象？

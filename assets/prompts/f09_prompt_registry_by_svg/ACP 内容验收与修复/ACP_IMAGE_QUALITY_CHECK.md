ACP_IMAGE_QUALITY_CHECK

【定位】
- version：2.2.0
- phase：NodeExecutionResult -> Quality evaluation
- canonical semantics：`acp_quality_check_input.v1 -> quality_report.v1`；输入是 ACP 图像验收的阶段裁剪，评分输出仍使用统一 `quality_report.v1`
- input schema：acp_quality_check_input.v1
- output schema：quality_report.v1
- upstream producer：Runtime quality-context assembler
- downstream consumer：Runtime terminal handling；仅 repairable fail 可进入一次性 Repair

【唯一职责】
以统一质量语义评估当前 image 节点的实际执行结果，并输出唯一 `quality_report.v1`。阈值、状态、finding、hard_fail 和 repairable 全部由当前 `quality_policy` 决定。

【明确不负责】
- 不重新 Planning、Routing、Binding，不改变 node_id/route_id，不调用 Provider。
- 不重写内容、不修改媒体、不生成替代 Provider Prompt、不选择 model/endpoint。
- 不根据 URL、文件名、标签、Prompt 描述或单帧推断未观察内容。
- 不维护旧 ACP pass key、固定 70 分阈值或另一套 checks schema。

【上游直接输出】
Runtime 提供 node/expected_outputs、相关 must/must_not/acceptance、Provider Prompt 有界证据、当前执行结果的最小证据切片、可观察图像证据与统一 quality_policy。

【Runtime 调用前组装】
1. 只组装当前图像节点、相关要求、实际执行结果、可观察证据和质量策略。
2. `node_id`、`route_id` 只在 node 与报告中保留一次，用于关联评估对象；Runtime 在模型调用外把 execution evidence 绑定到该节点并校验 Binding 归属。
3. 只注入当前节点相关要求和证据，不注入完整聊天、凭据、真实 URL、费用策略或无关节点输出。
4. observable_evidence 必须声明覆盖和限制；`attempt` 与 `quality_policy.max_repair_attempts` 共同决定修复预算是否仍可用。
5. quality_policy 是 pass/warn 阈值与最多修复次数的唯一来源，不沿用旧 ACP 固定阈值。

【严格输入 JSON】
```json
{
  "schema_version": "acp_quality_check_input.v1",
  "prompt_call": {
    "prompt_id": "ACP_IMAGE_QUALITY_CHECK",
    "prompt_version": "2.2.0",
    "attempt": 0
  },
  "node": {
    "node_id": "n20",
    "route_id": "R03",
    "objective": "生成一张用于产品发布演示的 16:9 宽屏封面底图，并保留中央标题安全区",
    "expected_outputs": [
      {
        "key": "image",
        "type": "image",
        "cardinality": "single",
        "quantity": 1
      }
    ]
  },
  "requirements": {
    "must": [
      "主体与封面用途一致",
      "中央标题安全区可用"
    ],
    "must_not": [
      "不得出现乱码或随机 Logo"
    ],
    "acceptance": [
      "图像清晰且没有明显生成伪影"
    ]
  },
  "provider_prompt_evidence": {
    "summary": "bounded provider prompt summary",
    "package_ref": "provider-prompt-package-ref"
  },
  "execution_evidence": {
    "schema_version": "node_execution_evidence.v1",
    "status": "succeeded",
    "artifacts": [
      {
        "artifact_ref": "image-001",
        "kind": "image"
      }
    ],
    "technical_metadata": {
      "width": 1280,
      "height": 720,
      "mime_type": "image/png"
    },
    "evidence_refs": [
      "image-bytes-001",
      "reference:image-001"
    ]
  },
  "observable_evidence": {
    "artifacts": [
      {
        "evidence_ref": "image-bytes-001",
        "artifact_ref": "image-001",
        "coverage": "full"
      }
    ],
    "references": [
      {
        "input_id": "reference-image-001",
        "role": "style",
        "analysis_ref": "reference:image-001"
      }
    ],
    "metadata": {
      "width": 1280,
      "height": 720
    },
    "coverage_limitations": []
  },
  "quality_policy": {
    "pass_threshold": 85,
    "warn_threshold": 70,
    "max_repair_attempts": 1,
    "required_dimensions": [
      "technical",
      "semantic_match",
      "composition",
      "reference_preservation",
      "text_artifacts",
      "delivery_safety"
    ],
    "dimension_weights": {
      "technical": 0.1,
      "semantic_match": 0.2,
      "composition": 0.4,
      "reference_preservation": 0.1,
      "text_artifacts": 0.1,
      "delivery_safety": 0.1
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
7. repairable 仅在 max_repair_attempts=1、非 hard_fail、failed finding 有证据且同一 Rewriter 可定向修正时为 true；repair_context 只列失败 finding_id。
8. 必须实际使用 observable_evidence 中可访问的图片证据；只有 Prompt 摘要或 URL 时返回 not_evaluable。
9. technical 依据解码/metadata；semantic_match 对照 node、requirements 和 provider_prompt_evidence；reference_preservation 只依据真实 reference analysis/evidence。
10. 检查主体、场景、构图、景别、视角、景深、光色、材质、文字安全区、乱码、随机标识、畸变、重复和边缘伪影。
11. 不从文件名、标签或 prompt 声称推断画面事实；证据不足的维度必须使用 `status=not_evaluable, score=0` 并形成 finding，不得输出 schema 之外的 `evaluable` 字段。
12. dimensions 固定表达 technical、semantic_match、composition、reference_preservation、text_artifacts、delivery_safety。
13. 只有可由同一 Rewriter 定向约束修正、且修复预算未耗尽的 failed finding 才令 repairable=true。

【严格输出 JSON】
只输出一个可解析 JSON 对象，不输出 Markdown、代码围栏或解释；顶层恰好为 schema_version、status、node_id、route_id、score、hard_fail、dimensions、findings、repairable、repair_context、provenance：

```json
{
  "schema_version": "quality_report.v1",
  "status": "fail",
  "node_id": "n20",
  "route_id": "R03",
  "score": 79,
  "hard_fail": false,
  "dimensions": {
    "technical": {
      "score": 96,
      "status": "pass",
      "evidence_refs": [
        "image-bytes-001"
      ],
      "notes": "图像可解码且尺寸符合"
    },
    "semantic_match": {
      "score": 88,
      "status": "pass",
      "evidence_refs": [
        "image-bytes-001"
      ],
      "notes": "主体与封面用途匹配"
    },
    "composition": {
      "score": 62,
      "status": "fail",
      "evidence_refs": [
        "image-bytes-001"
      ],
      "notes": "标题安全区被主体遮挡"
    },
    "reference_preservation": {
      "score": 90,
      "status": "pass",
      "evidence_refs": [
        "image-bytes-001",
        "reference:image-001"
      ],
      "notes": "可观察参考属性得到保留"
    },
    "text_artifacts": {
      "score": 95,
      "status": "pass",
      "evidence_refs": [
        "image-bytes-001"
      ],
      "notes": "未观察到明显乱码或随机标识"
    },
    "delivery_safety": {
      "score": 86,
      "status": "pass",
      "evidence_refs": [
        "image-bytes-001"
      ],
      "notes": "未观察到硬性交付风险"
    }
  },
  "findings": [
    {
      "finding_id": "img-f01",
      "code": "title_safe_area_obstructed",
      "severity": "error",
      "requirement_ref": "/requirements/must/1",
      "evidence_ref": "image-bytes-001",
      "description": "中央标题安全区被高对比主体占用，无法可靠叠加标题。"
    }
  ],
  "repairable": true,
  "repair_context": {
    "failed_finding_refs": [
      "img-f01"
    ]
  },
  "provenance": {
    "prompt_id": "ACP_IMAGE_QUALITY_CHECK",
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
Runtime 直接消费 canonical status。pass/warn 进入相应交付策略；不可修 fail/not_evaluable 终止或显式失败。仅对 repairable fail，Runtime 在模型外保存完整原始 Rewriter 输入，并从本 `quality_report.v1` 确定性裁剪 `quality_failure_slice.v1`，逐字保留当前 finding、所需维度及 requirement/evidence refs；再把该 slice、`rewriter_context`、Prompt 片段与图像证据交给 ACP Image Repair。Repair 产出 `rewriter_correction_context.v1` 后，Runtime 校验并注入完整原输入，再让同一个 Image Rewriter 重跑一次（attempt=1/max_attempts=1）。

【自检】
1. 是否逐字复制 node_id/route_id，并只使用当前可观察证据？
2. 是否使用 quality_policy，而非旧 ACP 固定阈值或 pass key？
3. 是否没有把 package 描述、单帧或无音频证据冒充完整媒体事实？
4. findings 是否可定位，status/score/hard_fail/repairable 是否一致？
5. 是否只输出一个 quality_report.v1 JSON 对象？

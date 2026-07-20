PACKAGE_QUALITY_GATE

【定位】
- phase：Phase 6 Generic Quality。
- version：2.1.0。
- contract：`quality_gate_input.v1 → quality_report.v1`。
- canonical role：TO-BE package 聚合质量权威 evaluator。

【唯一职责】
评估包结构、文本、引用完整性和 child quality reports，并形成 package 级结论。

【明确不负责】
不假装直接观看所有 child 媒体，不覆盖 child evaluator 的媒体结论，不修改包、不重新渲染、不路由、不调用 Provider。

【上游直接输出】
Package 的标准化 execution result、可观察包结构/文本，以及每个 required child 的 `quality_report.v1`。

【Runtime 调用前组装】
Runtime 只组装当前包节点的要求、包结构证据、required child 清单、child reports 和质量策略。当前包与 child 的 `node_id`、`route_id` 用于核对 required slot、报告归属和状态传播；Binding 归属由 Runtime 在模型调用外校验。当前没有 package correction consumer，因此不注入修复次数或修复预算。Runtime 为每份 child report 提供稳定 evidence ref，不注入无关节点。

【严格输入 JSON】

```json
{
  "schema_version": "quality_gate_input.v1",
  "prompt_call": {
    "prompt_id": "PACKAGE_QUALITY_GATE",
    "prompt_version": "2.1.0"
  },
  "node": {
    "expected_outputs": [
      {
        "key": "content_package",
        "type": "presentation_content_package",
        "cardinality": "single",
        "quantity": 1
      }
    ],
    "required_child_nodes": [
      {
        "node_id": "n02",
        "route_id": "R03",
        "slot_id": "asset-01"
      },
      {
        "node_id": "n03",
        "route_id": "R05",
        "slot_id": "asset-02"
      }
    ]
  },
  "requirements": {
    "must": [
      {
        "ref": "/requirements/must/0",
        "text": "两页内容和两个媒体 slot 完整"
      }
    ],
    "must_not": [],
    "acceptance": [
      {
        "ref": "/requirements/acceptance/0",
        "text": "所有 required child 均通过质量检查"
      }
    ]
  },
  "execution_result": {
    "schema_version": "node_execution_result.v1",
    "node_id": "n01",
    "route_id": "R01",
    "status": "succeeded",
    "artifacts": [
      {
        "artifact_id": "pkg-001",
        "kind": "presentation_package"
      }
    ],
    "technical_metadata": {
      "page_count": 2
    },
    "evidence_refs": [
      "package:structure",
      "package:text"
    ]
  },
  "observable_evidence": {
    "artifacts": [
      {
        "artifact_id": "pkg-001",
        "kind": "presentation_package",
        "structure": {
          "page_count": 2,
          "slot_ids": [
            "asset-01",
            "asset-02"
          ]
        },
        "evidence_ref": "package:structure"
      }
    ],
    "metadata": {
      "package_text_ref": "package:text"
    },
    "coverage": {
      "package_structure": "full",
      "package_text": "full",
      "media_content": "delegated_to_child_reports"
    },
    "coverage_limitations": [
      "package evaluator does not directly inspect child media bytes"
    ]
  },
  "child_quality_reports": [
    {
      "report_ref": "child-report:n02",
      "schema_version": "quality_report.v1",
      "status": "pass",
      "node_id": "n02",
      "route_id": "R03",
      "score": 91,
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
      "structure_contract",
      "cross_reference_integrity",
      "requirement_coverage",
      "child_quality_aggregation",
      "delivery_readiness"
    ],
    "dimension_weights": {
      "structure_contract": 0.25,
      "cross_reference_integrity": 0.2,
      "requirement_coverage": 0.2,
      "child_quality_aggregation": 0.25,
      "delivery_readiness": 0.1
    }
  }
}
```

【评估维度】
- `structure_contract`：页/文件/section/sheet 等声明结构、数量和类型。
- `cross_reference_integrity`：slot、artifact、文本、表格和 child node 引用可解析。
- `requirement_coverage`：package 层 must/must_not/acceptance 覆盖。
- `child_quality_aggregation`：所有 required child reports 的完整性和状态传播。
- `delivery_readiness`：在不重复评估媒体内容的前提下，包是否可进入 renderer/交付。

【证据规则】
Package evaluator 可直接判断包结构、文本和引用；媒体语义/画质/连续性/音质只能引用对应 child report。required child 缺报告时不能推断通过。child `hard_fail` 必须传播为 package hard fail；child fail 至少传播为 package fail；child warn 令 package 最多为 warn；required child not_evaluable 令 package 不可整体通过。

【统一判定语义】
1. 阈值满足 `0 <= warn_threshold <= pass_threshold <= 100`，权重总和为 1。
2. 全部 required dimensions 可评估时计算加权整数 score；维度阈值语义与总阈值相同。
3. hard_fail finding 或 child hard_fail 令 `hard_fail=true,status=fail`；error finding、child fail 或 score 低于 warn threshold 令 fail；其余情况下有 warning/child warn/score 低于 pass threshold 令 warn，否则 pass。
4. hard_fail 必须有 severity=`hard_fail` finding；来自 child 时创建一个指向 child report 的聚合 finding。
5. 当前没有 package correction consumer，`repairable` 固定为 false、`repair_context` 固定为 null；child 媒体失败也不得冒充 package 可修。
6. finding severity 只用 `warning|error|hard_fail`，不得复制 child 的全部 finding；只生成可追溯的聚合 finding。

【严格输出 JSON】

只输出一个 JSON 对象，不输出 Markdown、代码围栏或解释。

```json
{
  "schema_version": "quality_report.v1",
  "status": "not_evaluable",
  "node_id": "n01",
  "route_id": "R01",
  "score": 0,
  "hard_fail": false,
  "dimensions": {
    "structure_contract": {
      "score": 95,
      "status": "pass",
      "evidence_refs": [
        "package:structure"
      ],
      "notes": "两页与两个 slot 均存在"
    },
    "cross_reference_integrity": {
      "score": 90,
      "status": "pass",
      "evidence_refs": [
        "package:structure"
      ],
      "notes": "可见引用有效"
    },
    "requirement_coverage": {
      "score": 88,
      "status": "pass",
      "evidence_refs": [
        "package:text"
      ],
      "notes": "包层要求已覆盖"
    },
    "child_quality_aggregation": {
      "score": 0,
      "status": "not_evaluable",
      "evidence_refs": [
        "child-report:n02"
      ],
      "notes": "required child n03 缺少报告"
    },
    "delivery_readiness": {
      "score": 0,
      "status": "not_evaluable",
      "evidence_refs": [
        "package:structure"
      ],
      "notes": "无法确认全部 child 可交付"
    }
  },
  "findings": [
    {
      "finding_id": "f01",
      "code": "missing_required_child_report",
      "severity": "warning",
      "requirement_ref": "/requirements/acceptance/0",
      "evidence_ref": "package:structure",
      "description": "required child n03 没有 quality_report.v1，不能判断其视频质量"
    }
  ],
  "repairable": false,
  "repair_context": null,
  "provenance": {
    "prompt_id": "PACKAGE_QUALITY_GATE",
    "prompt_version": "2.1.0"
  }
}
```

【not_evaluable】
包结构/正文不可观察，required child report 缺失、child node/route 不匹配或为 not_evaluable，且现有证据不足以形成整体结论时，整体为 `not_evaluable,score=0`。补报告是 Runtime 证据任务，不是 package Rewriter repair。

【下游消费方式】
Runtime 依据 package 状态决定交付、记录风险、上层重新规划或补齐 child evaluation；在 package correction consumer 接线前不得宣称可定向修复，也不得用 package Gate 替代媒体 Gate。

【自检】
核对当前包与 required child 的节点关联、required child 覆盖、状态传播、阈值/权重和聚合 finding；确认未声称直接观看 child 媒体。

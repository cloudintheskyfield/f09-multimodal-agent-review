ACP_VIDEO_QUALITY_CHECK

【定位】
- version：2.2.0
- phase：NodeExecutionResult -> Quality evaluation
- canonical semantics：`acp_quality_check_input.v1 -> quality_report.v1`；输入是 ACP 视频验收的阶段裁剪，评分输出仍使用统一 `quality_report.v1`
- input schema：acp_quality_check_input.v1
- output schema：quality_report.v1
- upstream producer：Runtime quality-context assembler
- downstream consumer：Runtime terminal handling；仅 repairable fail 可进入一次性 Repair

【唯一职责】
以统一质量语义评估当前 video 节点的实际执行结果，并输出唯一 `quality_report.v1`。阈值、状态、finding、hard_fail 和 repairable 全部由当前 `quality_policy` 决定。

【明确不负责】
- 不重新 Planning、Routing、Binding，不改变 node_id/route_id，不调用 Provider。
- 不重写内容、不修改媒体、不生成替代 Provider Prompt、不选择 model/endpoint。
- 不根据 URL、文件名、标签、Prompt 描述或单帧推断未观察内容。
- 不维护旧 ACP pass key、固定 70 分阈值或另一套 checks schema。

【上游直接输出】
Runtime 提供 node/expected_outputs、相关 must/must_not/acceptance、Provider Prompt 有界证据、当前执行结果的最小证据切片、可观察视频证据与统一 quality_policy。

【Runtime 调用前组装】
1. 只组装当前视频节点、相关要求、实际执行结果、可观察证据和质量策略。
2. `node_id`、`route_id` 只在 node 与报告中保留一次，用于关联评估对象；Runtime 在模型调用外把 execution evidence 绑定到该节点并校验 Binding 归属。
3. 只注入当前节点相关要求和证据，不注入完整聊天、凭据、真实 URL、费用策略或无关节点输出。
4. observable_evidence 必须声明视频、帧、音轨与转写的真实覆盖；`attempt` 与 `quality_policy.max_repair_attempts` 共同决定修复预算是否仍可用。
5. quality_policy 是 pass/warn 阈值与最多修复次数的唯一来源，不沿用旧 ACP 固定阈值。

【严格输入 JSON】
```json
{
  "schema_version": "acp_quality_check_input.v1",
  "prompt_call": {
    "prompt_id": "ACP_VIDEO_QUALITY_CHECK",
    "prompt_version": "2.2.0",
    "attempt": 0
  },
  "node": {
    "node_id": "n30",
    "route_id": "R05",
    "objective": "生成一段 8 秒竖屏产品展示视频，动作连续，并保留稳定的 Logo 终帧安全区",
    "expected_outputs": [
      {
        "key": "video",
        "type": "video",
        "cardinality": "single",
        "quantity": 1
      }
    ]
  },
  "requirements": {
    "must": [
      "总时长 8 秒",
      "终帧稳定且保留 Logo 安全区"
    ],
    "must_not": [
      "不得出现主体漂移"
    ],
    "acceptance": [
      "动作和镜头连续，无闪烁或突兀切镜"
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
        "artifact_ref": "video-001",
        "kind": "video"
      }
    ],
    "technical_metadata": {
      "duration_ms": 8000,
      "width": 720,
      "height": 1280,
      "mime_type": "video/mp4"
    },
    "evidence_refs": [
      "video-decode-001",
      "frame-end",
      "audio:full"
    ]
  },
  "observable_evidence": {
    "artifacts": [
      {
        "evidence_ref": "video-decode-001",
        "artifact_ref": "video-001",
        "coverage": "full_decode"
      }
    ],
    "representative_frames": [
      {
        "evidence_ref": "frame-start",
        "time_ms": 0
      },
      {
        "evidence_ref": "frame-middle",
        "time_ms": 4000
      },
      {
        "evidence_ref": "frame-end",
        "time_ms": 7900
      }
    ],
    "transcript": [],
    "audio_observations": [
      {
        "evidence_ref": "audio:full",
        "coverage": "full",
        "description": "全段音轨可观察；本节点没有口播或字幕要求"
      }
    ],
    "metadata": {
      "duration_ms": 8000,
      "frame_coverage": "full"
    },
    "coverage_limitations": []
  },
  "quality_policy": {
    "pass_threshold": 85,
    "warn_threshold": 70,
    "max_repair_attempts": 1,
    "required_dimensions": [
      "technical",
      "content_match",
      "temporal_continuity",
      "motion",
      "text_audio",
      "ending",
      "delivery_safety"
    ],
    "dimension_weights": {
      "technical": 0.1,
      "content_match": 0.15,
      "temporal_continuity": 0.1,
      "motion": 0.1,
      "text_audio": 0.1,
      "ending": 0.35,
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
8. 技术与全片结论必须由实际视频解码、metadata、时间覆盖、代表帧、转写或音频证据支持。
9. 代表帧只能证明对应时点；覆盖不足时不得宣称整段连续、无闪烁或音画同步，必要时返回 not_evaluable。
10. 检查时长/比例/帧完整性、内容匹配、主体与材质连续性、运动路径/速度/惯性、运镜/转场、文字伪影、音画策略和稳定终帧。
11. transcript 只证明其覆盖范围；没有音频证据时 `text_audio` 维度必须使用 `status=not_evaluable, score=0`，不得猜声音或输出 schema 之外的 `evaluable` 字段。
12. dimensions 固定表达 technical、content_match、temporal_continuity、motion、text_audio、ending、delivery_safety。
13. 只有 failed finding 有明确时间/evidence、同一 Rewriter 可定向修正且修复预算未耗尽时，才令 repairable=true。

【严格输出 JSON】
只输出一个可解析 JSON 对象，不输出 Markdown、代码围栏或解释；顶层恰好为 schema_version、status、node_id、route_id、score、hard_fail、dimensions、findings、repairable、repair_context、provenance：

```json
{
  "schema_version": "quality_report.v1",
  "status": "fail",
  "node_id": "n30",
  "route_id": "R05",
  "score": 78,
  "hard_fail": false,
  "dimensions": {
    "technical": {
      "score": 95,
      "status": "pass",
      "evidence_refs": [
        "video-decode-001"
      ],
      "notes": "视频可解码且技术参数可用"
    },
    "content_match": {
      "score": 87,
      "status": "pass",
      "evidence_refs": [
        "video-decode-001"
      ],
      "notes": "可观察内容与节点目标匹配"
    },
    "temporal_continuity": {
      "score": 84,
      "status": "warn",
      "evidence_refs": [
        "video-decode-001"
      ],
      "notes": "主体与场景在已覆盖区间保持连续"
    },
    "motion": {
      "score": 82,
      "status": "warn",
      "evidence_refs": [
        "video-decode-001"
      ],
      "notes": "已覆盖区间的运动基本自然"
    },
    "text_audio": {
      "score": 90,
      "status": "pass",
      "evidence_refs": [
        "audio:full"
      ],
      "notes": "全段音轨可观察，且本节点没有口播或字幕要求"
    },
    "ending": {
      "score": 61,
      "status": "fail",
      "evidence_refs": [
        "video-decode-001",
        "frame-end"
      ],
      "notes": "结束阶段仍有运动，终帧不稳定"
    },
    "delivery_safety": {
      "score": 86,
      "status": "pass",
      "evidence_refs": [
        "video-decode-001"
      ],
      "notes": "未观察到其他硬性交付风险"
    }
  },
  "findings": [
    {
      "finding_id": "vid-f01",
      "code": "unstable_end_frame",
      "severity": "error",
      "requirement_ref": "/requirements/must/1",
      "evidence_ref": "video-decode-001",
      "description": "结束阶段仍有明显镜头运动，未形成可叠加 Logo 的稳定终帧。"
    }
  ],
  "repairable": true,
  "repair_context": {
    "failed_finding_refs": [
      "vid-f01"
    ]
  },
  "provenance": {
    "prompt_id": "ACP_VIDEO_QUALITY_CHECK",
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
Runtime 直接消费 canonical status。pass/warn 进入相应交付策略；不可修 fail/not_evaluable 终止或显式失败。仅对 repairable fail，Runtime 在模型外保存完整原始 Rewriter 输入，并从本 `quality_report.v1` 确定性裁剪 `quality_failure_slice.v1`，逐字保留当前 finding、所需维度及 requirement/evidence refs；再把该 slice、`rewriter_context`、Prompt 片段与时序证据交给 ACP Video Repair。Repair 产出 `rewriter_correction_context.v1` 后，Runtime 校验并注入完整原输入，再让同一个 Video Rewriter 重跑一次（attempt=1/max_attempts=1）。

【自检】
1. 是否逐字复制 node_id/route_id，并只使用当前可观察证据？
2. 是否使用 quality_policy，而非旧 ACP 固定阈值或 pass key？
3. 是否没有把 package 描述、单帧或无音频证据冒充完整媒体事实？
4. findings 是否可定位，status/score/hard_fail/repairable 是否一致？
5. 是否只输出一个 quality_report.v1 JSON 对象？

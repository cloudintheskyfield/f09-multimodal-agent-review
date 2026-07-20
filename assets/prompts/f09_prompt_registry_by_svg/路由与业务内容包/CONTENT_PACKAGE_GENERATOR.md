CONTENT_PACKAGE_GENERATOR

【定位】
- phase：Phase 3，R01 复合父节点的内容包节点。
- prompt family：business/presentation-package。
- compatible routes：R01。
- version：3.2.0。
- input schema：`resolved_node_context.v1`。
- output schema：`business_node_result.v1`。
- upstream producer：受保护抽象 DAG 经 Runtime 确定性 Rxx resolver 后形成的 R01 父节点；Runtime 在模型外保存 child/Route/selector 拓扑，只向模型投影语义化 `required_slots`。
- downstream consumer：presentation renderer；Runtime 按 `slot_id` 重新附加 child/Route/selector 后把 `asset_plan.items` 交给对应媒体子节点。

【唯一职责】
生成页面文案、讲稿和创意素材 brief，并精确填满 Runtime 给定的每一个 `required_slots`。

【明确不负责】
- 不决定页外 child 数量，不新增/删除/重排 slot 或 DAG node；模型不接收也不回传 child node、Route 或 selector。
- 不生成 Provider Prompt、provider 参数、真实媒体或演示文件。
- 不选择 binding/provider/model/endpoint；不把 creative brief 描述成已生成素材。

【上游直接输出】
Runtime routed-DAG resolver 输出的 R01 父节点；child 拓扑留在 Runtime，只有每个素材位的语义用途进入本 Prompt。上游分析只作为 resolved input/upstream output 进入。

【Runtime 调用前组装】
Runtime 解析 TaskBrief refs、父节点输入与 `node.expected_outputs`，并从 child nodes 生成有序 `required_slots`。模型只看到每个 slot 的 `slot_id`、`expected_output_type` 与 `purpose`；child node、Route 和 selector 始终留在 Runtime。调用模型前，Runtime 在模型外断言 slot 数量、类型与完整拓扑一致；模型只按 `slot_id` 填写创意内容。
`node_id` 与 `route_id` 仅用于当前父节点合同校验和输出关联；`prompt_id` 与 `prompt_version` 仅用于 schema/provenance 校验，不作为内容推理证据。

【严格输入 JSON】

```json
{
  "schema_version": "resolved_node_context.v1",
  "prompt_call": {
    "prompt_id": "CONTENT_PACKAGE_GENERATOR",
    "prompt_version": "3.2.0",
    "node_id": "n01",
    "route_id": "R01"
  },
  "node": {
    "title": "生成演示内容包",
    "objective": "生成两页演示文案及两个静态素材 slot",
    "expected_outputs": [
      {
        "key": "content_package",
        "type": "presentation_content_package",
        "cardinality": "single",
        "quantity": 1,
        "description": "页面文案与讲稿"
      },
      {
        "key": "asset_plan",
        "type": "asset_plan",
        "cardinality": "collection",
        "quantity": 2,
        "description": "与 required slots 对齐的素材 brief"
      }
    ],
    "requirement_refs": [
      "/requirements/must/0"
    ],
    "outcome_refs": [
      "/outcomes/0"
    ]
  },
  "resolved_context": {
    "task": {
      "summary": "制作低碳转型两页演示内容",
      "subject": "低碳转型",
      "audience": "管理层",
      "purpose": "支持内部评审"
    },
    "outcomes": [
      {
        "outcome_ref": "/outcomes/0",
        "description": "两页演示内容包",
        "format": "presentation",
        "quantity": 1
      }
    ],
    "requirements": {
      "must": [
        {
          "ref": "/requirements/must/0",
          "text": "两页分别说明现状与行动"
        }
      ],
      "prefer": [],
      "must_not": [],
      "acceptance": []
    },
    "context_facts": []
  },
  "resolved_inputs": [],
  "upstream_outputs": [],
  "downstream_contract": {
    "required_slots": [
      {
        "slot_id": "asset-01",
        "expected_output_type": "image",
        "purpose": "第一页现状视觉"
      },
      {
        "slot_id": "asset-02",
        "expected_output_type": "video",
        "purpose": "第二页行动动态视觉"
      }
    ]
  },
  "runtime_policy": {
    "output_language": "zh-CN",
    "strict_json": true,
    "allow_defaults": false,
    "max_output_chars": 16000
  }
}
```

【处理规则】
1. `node.expected_outputs` 是唯一输出合同；`outputs` 只能包含其中声明的同名/同类型 key，且 key 必须逐字复制。
2. `asset_plan.items` 的数量和顺序必须与 Runtime 已校验的 `required_slots` 完全一致。逐字复制 `slot_id`、`expected_output_type`、`purpose`，不生成 child node、route_id 或 selector；Runtime 在模型外重新附加并校验拓扑。
3. 每个 slot 只填写 `creative_brief`、`source_input_refs`、`requirement_refs`、`acceptance_criteria`；creative brief 是节点意图，不是 Provider Prompt。
4. slides[].slot_ids 只能引用 required_slots 中的 slot_id；准确文字、Logo、图表标签交给 renderer/programmatic overlay。
5. 无证据事实使用限定表达并写入 `fact_check_items`。

【严格输出 JSON】

只输出一个 JSON 对象，不输出 Markdown、代码围栏或解释。

```json
{
  "schema_version": "business_node_result.v1",
  "status": "ready",
  "reason": "content package is complete and every immutable asset slot is filled once",
  "node_id": "n01",
  "route_id": "R01",
  "outputs": {
    "content_package": {
      "title": "低碳转型行动方案",
      "language": "zh-CN",
      "audience": "管理层",
      "objective": "说明现状与行动路径",
      "style": "理性、现代、克制",
      "slides": [
        {
          "slide_no": 1,
          "title": "先看清现状",
          "layout": "left_text_right_media",
          "headline": "用统一口径建立可复核基线",
          "body": [
            "明确数据边界、时间范围和责任来源。"
          ],
          "speaker_notes": "说明基线是排列后续行动优先级的前提。",
          "slot_ids": [
            "asset-01"
          ],
          "data_or_map_suggestions": [],
          "acceptance_criteria": [
            "主观点清楚",
            "素材不承载必须准确绘制的正文"
          ]
        },
        {
          "slide_no": 2,
          "title": "再推动行动",
          "layout": "full_bleed_media_with_overlay",
          "headline": "从高影响、可执行环节开始",
          "body": [
            "依据影响、难度和数据完备度确定先后顺序。"
          ],
          "speaker_notes": "强调行动顺序需要在数据核验后确认。",
          "slot_ids": [
            "asset-02"
          ],
          "data_or_map_suggestions": [],
          "acceptance_criteria": [
            "行动逻辑连续",
            "保留程序化文字安全区"
          ]
        }
      ],
      "fact_check_items": [],
      "renderer_handoff": {
        "renderer_kind": "presentation",
        "instructions": [
          "按 slide_no 排版",
          "只使用已验收 child artifacts 填充 slot"
        ]
      }
    },
    "asset_plan": {
      "items": [
        {
          "slot_id": "asset-01",
          "expected_output_type": "image",
          "purpose": "第一页现状视觉",
          "creative_brief": {
            "subject": "现代园区的能源数据基线示意",
            "scene": "清晰分层的园区空间，右侧主体，左侧留白",
            "composition": "wide establishing view with stable text-safe area",
            "style": "写实、现代、克制",
            "motion": null,
            "text_policy": "programmatic_overlay_only"
          },
          "source_input_refs": [],
          "requirement_refs": [
            "/requirements/must/0"
          ],
          "acceptance_criteria": [
            "主体层级清楚",
            "左侧安全区连续"
          ]
        },
        {
          "slot_id": "asset-02",
          "expected_output_type": "video",
          "purpose": "第二页行动动态视觉",
          "creative_brief": {
            "subject": "园区能源节点依次点亮",
            "scene": "夜色平滑过渡到清晨",
            "composition": "single continuous establishing shot",
            "style": "写实电影感、深蓝到暖金",
            "motion": "节点由远及近依次点亮，镜头缓慢推进并稳定结束",
            "text_policy": "programmatic_overlay_only"
          },
          "source_input_refs": [],
          "requirement_refs": [
            "/requirements/must/0"
          ],
          "acceptance_criteria": [
            "运动连续无瞬移",
            "终帧稳定并保留标题区"
          ]
        }
      ]
    }
  },
  "validation": {
    "fulfilled_output_keys": [
      "content_package",
      "asset_plan"
    ],
    "missing_output_keys": [],
    "warnings": []
  },
  "provenance": {
    "prompt_id": "CONTENT_PACKAGE_GENERATOR",
    "prompt_version": "3.2.0",
    "source": "llm"
  }
}
```

【状态与阻塞】
缺少 `node.expected_outputs`、slot 语义字段不完整、slot 数量/类型冲突或硬约束冲突时返回 `blocked` 且 `outputs={}`；不得自行修复 slot。child 拓扑错误由 Runtime 在模型外失败关闭。仅当 `node.expected_outputs` 为空时使用 `no_action`。

【下游消费方式】
Renderer 消费 content_package；Runtime 按 `slot_id` 为每个 asset_plan item 重新附加已保存的 selector 与 child/Route 绑定，再交给对应 Resolver/Rewriter 编译 Provider Prompt。

【自检】
检查输出 key、slot 数量/顺序、三个只读语义字段、slide-slot 引用和事实边界；确认没有接收或生成 child node、Route、selector、Provider Prompt 或已生成媒体声明。

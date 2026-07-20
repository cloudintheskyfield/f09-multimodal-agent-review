VIDEO_STORYBOARD_GENERATOR

【定位】
- phase：Phase 3，R02 复合父节点分镜规划。
- prompt family：business/storyboard-package。
- compatible routes：R02。
- version：2.2.0。
- input/output：`resolved_node_context.v1 → business_node_result.v1`。
- upstream：受保护抽象 DAG 经 Runtime 确定性 Rxx resolver 后形成的 R02 父节点；Runtime 保存 child/Route/selector 拓扑，只投影语义化 required_slots。
- downstream：renderer 使用 storyboard；Runtime 按 `slot_id` 重新附加 child/Route/selector 后把 shot item 交给对应视频子节点。

【唯一职责】
生成叙事、镜头、声音与连续性规格，并逐项填满 immutable required_slots。

【明确不负责】
不决定镜头/child 数量，不改 slot，不接收或回传 child node、Route、selector，不生成视频、Provider Prompt 或 endpoint 参数，不重新 Planning/Binding。

【上游直接输出】
Runtime routed-DAG resolver 输出的 R02 父节点；child 拓扑留在 Runtime，只有每个镜头位的语义用途进入本 Prompt。上游可观察媒体分析仅通过 Runtime 解析结果进入。

【Runtime 调用前组装】
Runtime 解析相关要求、输入分析、`node.expected_outputs`、总时长/比例等硬约束，并按 child node 顺序生成 `required_slots`。模型只看到 `slot_id`、`expected_output_type` 与 `purpose`；child node、Route 和 selector 始终留在 Runtime。调用模型前，Runtime 在模型外断言 slot 数量、类型与完整拓扑一致；缺失硬约束且 `allow_defaults=false` 时不得让模型猜测。
`node_id` 与 `route_id` 仅用于当前父节点合同校验和输出关联；`prompt_id` 与 `prompt_version` 仅用于 schema/provenance 校验。

【严格输入 JSON】

```json
{
  "schema_version": "resolved_node_context.v1",
  "prompt_call": {
    "prompt_id": "VIDEO_STORYBOARD_GENERATOR",
    "prompt_version": "2.2.0",
    "node_id": "n01",
    "route_id": "R02"
  },
  "node": {
    "title": "生成两镜头分镜",
    "objective": "用十秒短片说明从现状到行动的转变",
    "expected_outputs": [
      {
        "key": "storyboard",
        "type": "video_storyboard",
        "cardinality": "single",
        "quantity": 1,
        "description": "全片叙事与连续性"
      },
      {
        "key": "shot_plan",
        "type": "shot_plan",
        "cardinality": "collection",
        "quantity": 2,
        "description": "与 required slots 对齐的镜头 brief"
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
      "summary": "制作十秒低碳园区短片",
      "subject": "园区转型",
      "audience": "管理层",
      "purpose": "开场引题"
    },
    "outcomes": [
      {
        "outcome_ref": "/outcomes/0",
        "description": "十秒视频分镜",
        "format": "storyboard",
        "quantity": 1
      }
    ],
    "requirements": {
      "must": [
        {
          "ref": "/requirements/must/0",
          "text": "总时长十秒，16:9，两个镜头"
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
        "slot_id": "shot-01",
        "expected_output_type": "video",
        "purpose": "建立现状"
      },
      {
        "slot_id": "shot-02",
        "expected_output_type": "video",
        "purpose": "呈现行动"
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
1. `node.expected_outputs` 是唯一输出合同；`outputs` 只能包含其中声明的 key。`shot_plan.items` 与 Runtime 已校验的 `required_slots` 在数量、顺序、slot_id、类型和 purpose 上完全一致；Runtime 在模型外重新附加并校验 child/Route/selector 拓扑。
2. 每个 shot 给出 start/end、时间段、景别、机位、运镜、主体运动、声音、连续性、overlay 和验收标准；时间段无重叠/空洞，所有镜头时长之和等于总时长。
3. reference role 只来自输入分析；style reference 不自动成为 first frame。
4. shot creative_spec 仍是节点意图，不是 Provider Prompt。

【严格输出 JSON】

只输出一个 JSON 对象，不输出 Markdown、代码围栏或解释。

```json
{
  "schema_version": "business_node_result.v1",
  "status": "ready",
  "reason": "storyboard is complete and both immutable shot slots are filled",
  "node_id": "n01",
  "route_id": "R02",
  "outputs": {
    "storyboard": {
      "title": "从基线到行动",
      "objective": "用两个连续镜头建立转型主题",
      "duration_ms": 10000,
      "aspect_ratio": "16:9",
      "style_bible": "写实、克制；深蓝过渡到暖金；材质和光源方向稳定",
      "subject_bible": "园区建筑比例、设施轮廓和青绿色能源节点保持一致",
      "location_bible": "同一园区空间轴线，前景植被、中景设施、远景天际线",
      "shot_refs": [
        "shot-01",
        "shot-02"
      ],
      "global_continuity": [
        "空间方向一致",
        "主体比例与材质一致",
        "第二镜头从第一镜头稳定终帧承接"
      ],
      "audio_bible": "低频环境声持续，节点提示音与点亮节拍同步",
      "overlay_policy": "programmatic_only"
    },
    "shot_plan": {
      "items": [
        {
          "slot_id": "shot-01",
          "expected_output_type": "video",
          "purpose": "建立现状",
          "order": 1,
          "duration_ms": 5000,
          "creative_spec": {
            "start_frame": "夜色园区广角远景，能源节点未点亮",
            "timeline": [
              {
                "start_ms": 0,
                "end_ms": 5000,
                "action": "镜头沿中轴缓慢推进，保持现状氛围"
              }
            ],
            "shot_size": "wide",
            "camera": "略高平视，匀速直线推进并减速",
            "subject_motion": "建筑静止，少量环境光变化",
            "transition_out": "在同一轴线上稳定停留",
            "end_frame": "园区中景稳定，能源节点仍未点亮",
            "voiceover": "先看清每一次能源使用。",
            "audio": "低频环境声",
            "continuity": [
              "保持园区轮廓与光源方向"
            ],
            "text_policy": "programmatic_overlay_only"
          },
          "source_input_refs": [],
          "requirement_refs": [
            "/requirements/must/0"
          ],
          "acceptance_criteria": [
            "五秒完整",
            "无瞬移或闪烁",
            "终帧稳定"
          ]
        },
        {
          "slot_id": "shot-02",
          "expected_output_type": "video",
          "purpose": "呈现行动",
          "order": 2,
          "duration_ms": 5000,
          "creative_spec": {
            "start_frame": "承接上一镜头的园区中景",
            "timeline": [
              {
                "start_ms": 0,
                "end_ms": 5000,
                "action": "能源节点由远及近依次点亮，夜色平滑过渡到清晨"
              }
            ],
            "shot_size": "wide_to_medium",
            "camera": "沿同一轴线继续推进，末段平稳停止",
            "subject_motion": "能源流线匀速传播并自然减速",
            "transition_out": "无切镜，光色渐变结束",
            "end_frame": "清晨园区稳定停留，节点全部点亮",
            "voiceover": "再把数据变成有顺序的行动。",
            "audio": "环境声延续，节点提示音同步",
            "continuity": [
              "主体、材质、空间方向和运动方向连续"
            ],
            "text_policy": "programmatic_overlay_only"
          },
          "source_input_refs": [],
          "requirement_refs": [
            "/requirements/must/0"
          ],
          "acceptance_criteria": [
            "五秒完整",
            "动作连续",
            "稳定终帧"
          ]
        }
      ]
    }
  },
  "validation": {
    "fulfilled_output_keys": [
      "storyboard",
      "shot_plan"
    ],
    "missing_output_keys": [],
    "warnings": []
  },
  "provenance": {
    "prompt_id": "VIDEO_STORYBOARD_GENERATOR",
    "prompt_version": "2.2.0",
    "source": "llm"
  }
}
```

【状态与阻塞】
slot 语义字段不完整、总时长/slot 数量冲突、关键媒体引用缺失时 `blocked` 且 `outputs={}`；不得自行增减镜头。child/Route/selector 拓扑错误由 Runtime 在模型外失败关闭。

【下游消费方式】
Runtime 在模型外再次校验时长守恒与 slot 对齐，再按 `slot_id` 为每个 shot item 附加已保存的 child/Route/selector；Resolver 绑定后由对应 Rewriter 编译。

【自检】
检查 slot_id/类型/purpose 一致、时长守恒、时间无空洞、连续性/声音/终帧完整、引用存在，且没有接收或生成 child node、Route、selector、Provider Prompt 或新 DAG 节点。

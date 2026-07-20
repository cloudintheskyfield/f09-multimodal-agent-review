FIXED_LAYOUT_PDF_PACKAGE_GENERATOR

【定位】
- phase：Phase 3，固定版式 renderer spec 节点。
- family：business/pdf-renderer-spec。
- compatible routes：R16。
- version：3.1.0。
- contract：`resolved_node_context.v1 → business_node_result.v1`。
- downstream：确定性 PDF renderer 与 Package Quality Gate。

【唯一职责】
生成带页面设置、阅读顺序和固定区块几何的 PDF renderer spec；不生成真实 PDF。

【明确不负责】
不渲染文件、不创建媒体 child、不选 Provider/model/endpoint/adapter，不声称 artifact 已存在。

【上游直接输出】
当前 R16 node 与 Runtime 已解析的正文、表格、图表数据和 artifact refs。

【Runtime 调用前组装】
Runtime 仅注入当前节点相关要求、输入/上游输出和 `node.expected_outputs`；不注入密钥、真实 endpoint、无关记录。
`node_id` 与 `route_id` 仅用于当前节点合同校验和输出关联；`prompt_id` 与 `prompt_version` 仅用于 schema/provenance 校验。

【严格输入 JSON】

```json
{
  "schema_version": "resolved_node_context.v1",
  "prompt_call": {
    "prompt_id": "FIXED_LAYOUT_PDF_PACKAGE_GENERATOR",
    "prompt_version": "3.1.0",
    "node_id": "n16",
    "route_id": "R16"
  },
  "node": {
    "title": "生成 PDF 版式规格",
    "objective": "形成一页 A4 行动摘要",
    "expected_outputs": [
      {
        "key": "pdf_layout_spec",
        "type": "pdf_renderer_spec",
        "cardinality": "single",
        "quantity": 1,
        "description": "固定版式 renderer 输入"
      }
    ],
    "requirement_refs": [],
    "outcome_refs": [
      "/outcomes/0"
    ]
  },
  "resolved_context": {
    "task": {
      "summary": "制作一页行动摘要",
      "subject": "低碳行动",
      "audience": "管理层",
      "purpose": "打印评审"
    },
    "outcomes": [
      {
        "outcome_ref": "/outcomes/0",
        "description": "一页 PDF",
        "format": "pdf",
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
  "resolved_inputs": [],
  "upstream_outputs": [],
  "runtime_policy": {
    "output_language": "zh-CN",
    "strict_json": true,
    "allow_defaults": false,
    "max_output_chars": 18000
  }
}
```

【处理规则】
1. `target_format=pdf`，所有尺寸为 mm；区块必须在页面边界和 margins 内且不可重叠。
2. page_no 与 reading_order 连续；table/chart/asset refs 必须存在。
3. 图表没有可靠数据时保留空 data 并记录 source_requirement，不编造数值。
4. `renderer_handoff.file_generated=false`。

【严格输出 JSON】

只输出一个 JSON 对象，不输出 Markdown、代码围栏或解释。

```json
{
  "schema_version": "business_node_result.v1",
  "status": "ready",
  "reason": "fixed-layout PDF renderer specification is complete",
  "node_id": "n16",
  "route_id": "R16",
  "outputs": {
    "pdf_layout_spec": {
      "target_format": "pdf",
      "title": "低碳转型行动摘要",
      "language": "zh-CN",
      "page_setup": {
        "page_size": "A4",
        "width_mm": 210,
        "height_mm": 297,
        "orientation": "portrait",
        "margins_mm": {
          "top": 18,
          "right": 18,
          "bottom": 18,
          "left": 18
        }
      },
      "pages": [
        {
          "page_no": 1,
          "purpose": "概述目标与行动",
          "block_ids": [
            "blk-01",
            "blk-02"
          ],
          "reading_order": [
            "blk-01",
            "blk-02"
          ]
        }
      ],
      "blocks": [
        {
          "block_id": "blk-01",
          "page_no": 1,
          "block_type": "heading",
          "box_mm": {
            "x": 18,
            "y": 18,
            "width": 174,
            "height": 18
          },
          "content": "低碳转型行动路径",
          "object_ref": null,
          "style": {
            "font_size_pt": 20,
            "bold": true,
            "alignment": "left"
          }
        },
        {
          "block_id": "blk-02",
          "page_no": 1,
          "block_type": "paragraph",
          "box_mm": {
            "x": 18,
            "y": 44,
            "width": 174,
            "height": 45
          },
          "content": "先建立统一、可复核的能源基线，再依据业务影响、实施难度和数据完备度排列行动优先级。",
          "object_ref": null,
          "style": {
            "font_size_pt": 11,
            "bold": false,
            "alignment": "left"
          }
        }
      ],
      "tables": [],
      "charts": [],
      "asset_placements": [],
      "citations": [],
      "source_requirements": [],
      "accessibility": {
        "reading_order_explicit": true,
        "require_alt_text_for_assets": true
      },
      "renderer_handoff": {
        "renderer_kind": "pdf",
        "file_generated": false,
        "required_validations": [
          "all boxes are within page bounds",
          "blocks do not overlap",
          "all references resolve"
        ]
      }
    }
  },
  "validation": {
    "fulfilled_output_keys": [
      "pdf_layout_spec"
    ],
    "missing_output_keys": [],
    "warnings": []
  },
  "provenance": {
    "prompt_id": "FIXED_LAYOUT_PDF_PACKAGE_GENERATOR",
    "prompt_version": "3.1.0",
    "source": "llm"
  }
}
```

【状态与阻塞】
页面规格、必需内容或可解析引用缺失时 `blocked` 且 `outputs={}`；不得返回半成品或虚假文件 URL。

【下游消费方式】
PDF renderer 按坐标/阅读顺序生成文件；Runtime 标准化 renderer 结果后交质量环节。

【自检】
格式、`node_id`、`route_id` 和 output key 正确；几何在页面内、无重叠；引用和阅读顺序有效；没有虚构数据或文件完成声明。

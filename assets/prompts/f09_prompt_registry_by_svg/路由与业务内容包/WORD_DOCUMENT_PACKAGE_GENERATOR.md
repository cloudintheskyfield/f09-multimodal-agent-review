WORD_DOCUMENT_PACKAGE_GENERATOR

【定位】
- phase：Phase 3，文档 renderer spec 节点。
- family：business/document-renderer-spec。
- compatible routes：R15。
- version：3.1.0。
- contract：`resolved_node_context.v1 → business_node_result.v1`。
- downstream：确定性 DOCX renderer，然后 Package/Text Quality Gate。

【唯一职责】
生成 DOCX renderer 可消费的结构化文档规格；不生成真实 `.docx`。

【明确不负责】
不渲染/上传文件，不规划新媒体节点，不选择 Provider/model/endpoint/renderer 实现，不声称文件已生成。只引用 resolved input/upstream 中存在的资产。

【上游直接输出】
当前 R15 node，以及 Runtime 解析的文本、表格、来源、已完成媒体 artifact refs。

【Runtime 调用前组装】
Runtime 提供当前 node、相关要求、输入/上游内容和精确 output key；排除 credentials、真实 endpoint、无关历史和未引用资产。
`node_id` 与 `route_id` 仅用于当前节点合同校验和输出关联；`prompt_id` 与 `prompt_version` 仅用于 schema/provenance 校验。

【严格输入 JSON】

```json
{
  "schema_version": "resolved_node_context.v1",
  "prompt_call": {
    "prompt_id": "WORD_DOCUMENT_PACKAGE_GENERATOR",
    "prompt_version": "3.1.0",
    "node_id": "n15",
    "route_id": "R15"
  },
  "node": {
    "title": "生成 DOCX 规格",
    "objective": "生成行动方案文档 renderer spec",
    "expected_outputs": [
      {
        "key": "word_document_spec",
        "type": "docx_renderer_spec",
        "cardinality": "single",
        "quantity": 1,
        "description": "DOCX renderer 输入"
      }
    ],
    "requirement_refs": [],
    "outcome_refs": [
      "/outcomes/0"
    ]
  },
  "resolved_context": {
    "task": {
      "summary": "制作行动方案 Word 文档",
      "subject": "低碳行动",
      "audience": "管理层",
      "purpose": "评审"
    },
    "outcomes": [
      {
        "outcome_ref": "/outcomes/0",
        "description": "Word 文档",
        "format": "docx",
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
    "max_output_chars": 16000
  }
}
```

【处理规则】
1. 只输出 renderer spec；`target_format` 固定 `docx`，`renderer_handoff.file_generated=false`。
2. outline 与 sections 通过 section_id 一一对应；heading level 从 1 开始且不跳级。
3. table 行宽等于 columns 数；asset_placements 只引用真实 input/output/artifact ref。
4. 无来源声明写入 citations，状态 `needs_source`；不伪造引用。

【严格输出 JSON】

只输出一个 JSON 对象，不输出 Markdown、代码围栏或解释。

```json
{
  "schema_version": "business_node_result.v1",
  "status": "ready",
  "reason": "DOCX renderer specification is complete",
  "node_id": "n15",
  "route_id": "R15",
  "outputs": {
    "word_document_spec": {
      "target_format": "docx",
      "title": "低碳转型行动方案",
      "language": "zh-CN",
      "document_setup": {
        "page_size": "A4",
        "orientation": "portrait",
        "margins_mm": {
          "top": 20,
          "right": 20,
          "bottom": 20,
          "left": 20
        }
      },
      "style_tokens": [
        {
          "style_id": "title",
          "kind": "paragraph",
          "font_size_pt": 22,
          "bold": true,
          "alignment": "center"
        },
        {
          "style_id": "body",
          "kind": "paragraph",
          "font_size_pt": 10.5,
          "bold": false,
          "alignment": "left"
        }
      ],
      "outline": [
        {
          "section_id": "sec-01",
          "heading": "目标与范围",
          "level": 1,
          "purpose": "明确行动边界"
        }
      ],
      "sections": [
        {
          "section_id": "sec-01",
          "heading": "目标与范围",
          "level": 1,
          "blocks": [
            {
              "block_id": "blk-01",
              "block_type": "paragraph",
              "style_id": "body",
              "content": "本方案以建立可核验基线和分阶段行动清单为起点。",
              "table_ref": null,
              "asset_ref": null
            }
          ]
        }
      ],
      "tables": [],
      "asset_placements": [],
      "citations": [],
      "accessibility": {
        "heading_order_valid": true,
        "require_alt_text_for_assets": true
      },
      "renderer_handoff": {
        "renderer_kind": "docx",
        "file_generated": false,
        "required_validations": [
          "all references resolve",
          "heading hierarchy is valid"
        ]
      }
    }
  },
  "validation": {
    "fulfilled_output_keys": [
      "word_document_spec"
    ],
    "missing_output_keys": [],
    "warnings": []
  },
  "provenance": {
    "prompt_id": "WORD_DOCUMENT_PACKAGE_GENERATOR",
    "prompt_version": "3.1.0",
    "source": "llm"
  }
}
```

【状态与阻塞】
缺少必需正文、`node.expected_outputs` 未声明 `word_document_spec: docx_renderer_spec`、或引用不可解析时 `blocked` 且 `outputs={}`。不以“renderer 当前不可用”为由伪造文件。

【下游消费方式】
确定性 DOCX renderer 校验 spec 并生成真实文件；生成结果由 Runtime 标准化，质量环节检查结构和实际文件。

【自检】
输出 key、`node_id` 和 `route_id` 逐字一致；格式为 docx；层级、样式和引用有效；未生成资产计划、Provider 配置或文件完成声明。

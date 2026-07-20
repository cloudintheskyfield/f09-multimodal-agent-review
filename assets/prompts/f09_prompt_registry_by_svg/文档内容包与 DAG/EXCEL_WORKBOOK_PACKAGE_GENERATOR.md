EXCEL_WORKBOOK_PACKAGE_GENERATOR

【定位】
- phase：Phase 3，工作簿 renderer spec 节点。
- family：business/xlsx-renderer-spec。
- compatible routes：R17。
- version：3.1.0。
- contract：`resolved_node_context.v1 → business_node_result.v1`。
- downstream：确定性 XLSX renderer 与 Package Quality Gate。

【唯一职责】
生成可验证的 workbook schema、数据、公式、图表和数据校验规则；不生成真实 `.xlsx`。

【明确不负责】
不连接外部数据源、不编造缺失数值、不执行公式、不渲染文件、不选择 Provider/model/endpoint/renderer 实现。

【上游直接输出】
当前 R17 node 与 Runtime 解析的结构化数据、来源状态和上游业务结果。

【Runtime 调用前组装】
Runtime 注入当前节点相关 requirements、resolved inputs、upstream outputs 和 `node.expected_outputs`；不传凭据、真实 endpoint 或无关项目数据。
`node_id` 与 `route_id` 仅用于当前节点合同校验和输出关联；`prompt_id` 与 `prompt_version` 仅用于 schema/provenance 校验。

【严格输入 JSON】

```json
{
  "schema_version": "resolved_node_context.v1",
  "prompt_call": {
    "prompt_id": "EXCEL_WORKBOOK_PACKAGE_GENERATOR",
    "prompt_version": "3.1.0",
    "node_id": "n17",
    "route_id": "R17"
  },
  "node": {
    "title": "生成行动跟踪工作簿规格",
    "objective": "定义行动、完成数、总数和完成度字段",
    "expected_outputs": [
      {
        "key": "excel_workbook_spec",
        "type": "xlsx_renderer_spec",
        "cardinality": "single",
        "quantity": 1,
        "description": "XLSX renderer 输入"
      }
    ],
    "requirement_refs": [],
    "outcome_refs": [
      "/outcomes/0"
    ]
  },
  "resolved_context": {
    "task": {
      "summary": "制作行动跟踪表",
      "subject": "低碳行动",
      "audience": "项目团队",
      "purpose": "跟踪进度"
    },
    "outcomes": [
      {
        "outcome_ref": "/outcomes/0",
        "description": "Excel 工作簿",
        "format": "xlsx",
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
    "max_output_chars": 20000
  }
}
```

【处理规则】
1. `target_format=xlsx`；sheet/column/row/formula/chart/validation IDs 唯一且所有引用可解析。
2. data_type 只用 `string|integer|decimal|currency|percentage|date|datetime|boolean`。
3. 缺失值用 null，并登记 source_requirements；不得生成看似真实的示例业务数值。
4. 公式用 A1 表达式和结构化 dependencies；不得循环引用，除零必须处理。
5. 图表只引用已声明 range/column；renderer_handoff.file_generated=false。

【严格输出 JSON】

只输出一个 JSON 对象，不输出 Markdown、代码围栏或解释。

```json
{
  "schema_version": "business_node_result.v1",
  "status": "ready",
  "reason": "XLSX renderer specification is complete",
  "node_id": "n17",
  "route_id": "R17",
  "outputs": {
    "excel_workbook_spec": {
      "target_format": "xlsx",
      "title": "低碳行动跟踪表",
      "language": "zh-CN",
      "workbook": {
        "sheets": [
          {
            "sheet_id": "sheet-01",
            "name": "行动清单",
            "purpose": "记录行动与完成度",
            "column_ids": [
              "col-a",
              "col-b",
              "col-c",
              "col-d"
            ],
            "row_ids": [
              "row-01"
            ],
            "formula_ids": [
              "formula-01"
            ],
            "chart_ids": [
              "chart-01"
            ],
            "validation_ids": [
              "validation-01"
            ]
          }
        ]
      },
      "columns": [
        {
          "sheet_id": "sheet-01",
          "column_id": "col-a",
          "column_letter": "A",
          "name": "行动",
          "data_type": "string",
          "required": true,
          "number_format": null
        },
        {
          "sheet_id": "sheet-01",
          "column_id": "col-b",
          "column_letter": "B",
          "name": "已完成任务数",
          "data_type": "integer",
          "required": false,
          "number_format": "0"
        },
        {
          "sheet_id": "sheet-01",
          "column_id": "col-c",
          "column_letter": "C",
          "name": "任务总数",
          "data_type": "integer",
          "required": false,
          "number_format": "0"
        },
        {
          "sheet_id": "sheet-01",
          "column_id": "col-d",
          "column_letter": "D",
          "name": "完成度",
          "data_type": "percentage",
          "required": false,
          "number_format": "0%"
        }
      ],
      "rows": [
        {
          "sheet_id": "sheet-01",
          "row_id": "row-01",
          "row_number": 2,
          "values": {
            "col-a": "建立能源基线",
            "col-b": null,
            "col-c": null,
            "col-d": null
          },
          "source_status": "needs_source",
          "evidence_refs": []
        }
      ],
      "formulas": [
        {
          "sheet_id": "sheet-01",
          "formula_id": "formula-01",
          "target_cell": "D2",
          "expression": "=IFERROR(B2/C2,0)",
          "dependencies": [
            "B2",
            "C2"
          ],
          "result_type": "percentage",
          "validation_rule": "result must be between 0 and 1"
        }
      ],
      "charts": [
        {
          "sheet_id": "sheet-01",
          "chart_id": "chart-01",
          "chart_type": "bar",
          "title": "行动完成度",
          "category_range": "A2:A2",
          "value_ranges": [
            "D2:D2"
          ],
          "position": "F2:L18"
        }
      ],
      "data_validations": [
        {
          "validation_id": "validation-01",
          "sheet_id": "sheet-01",
          "range": "A2:A1000",
          "rule_type": "non_empty",
          "formula": null,
          "error_message": "行动名称不得为空"
        }
      ],
      "source_requirements": [
        {
          "requirement_id": "source-01",
          "target_refs": [
            "B2",
            "C2"
          ],
          "description": "补充已完成任务数与任务总数的可定位来源"
        }
      ],
      "renderer_handoff": {
        "renderer_kind": "xlsx",
        "file_generated": false,
        "required_validations": [
          "all references resolve",
          "formula graph is acyclic",
          "data types match cells"
        ]
      }
    }
  },
  "validation": {
    "fulfilled_output_keys": [
      "excel_workbook_spec"
    ],
    "missing_output_keys": [],
    "warnings": [
      "B2 and C2 require source data before final delivery"
    ]
  },
  "provenance": {
    "prompt_id": "EXCEL_WORKBOOK_PACKAGE_GENERATOR",
    "prompt_version": "3.1.0",
    "source": "llm"
  }
}
```

【状态与阻塞】
工作簿目标/必需列缺失、引用或公式无法无歧义构造时 `blocked` 且 `outputs={}`。缺少业务数据但可生成安全空模板时仍可 `ready`，必须用 null 和 source_requirements 标明。

【下游消费方式】
XLSX renderer 校验引用、类型和公式图后生成文件；Runtime 标准化结果，Quality Gate 检查真实 workbook。

【自检】
`node_id`、`route_id` 和 output key 逐字一致；引用存在、公式无环、缺失数据为 null、来源需求明确；无外部连接、Provider 配置或文件完成声明。

# MMAgent Prompt Registry

本目录是 multimodal-agent-diagram-collection.html 所列 41 个 Prompt 的独立设计注册表。HTML 决定 Prompt 的身份、顺序、状态和分组；Prompt 正文定义目标阶段合同，真实 Rust 类型、校验函数及调用位置用于记录当前运行证据。两者不一致时，Prompt 必须采用失败关闭规则，manifest 必须记录 `design_runtime_gap` 与 `required_runtime_assertions`，且不得把目标合同写成已部署事实。

本目录保留中文设计稿，供架构评审和中英文对照。模型实际调用应使用相邻的 `mmagent_prompt_registry_by_svg_en` 英文版；两套 Prompt 的职责、JSON 结构、枚举、版本和约束必须保持一致。生产加载器尚未切换到该独立注册表时，不得把“应使用英文版”描述成已经部署。

## 模型上下文最小化

- 本注册表只服务多媒体生成与内容包交付：文本内容、图片、视频、音频/语音、演示、Word、PDF、表格和多媒体内容包；理解、转写、内容规划、来源取得、事实核验与组装只能直接服务这些交付物。范围外请求使用 `unsupported_scope`，不得扩展成通用操作、代码或决策 Agent。
- Runtime 在模型调用外维护请求、会话、任务、用户、项目、幂等和事件追踪信息；这些全局关联字段不得进入 Prompt 的模型输入，也不得要求模型原样回传。
- Runtime 必须在每次调用前构造当前阶段的最小投影，只提供当前任务语义、允许使用的输入、可观察证据、当前节点约束和本阶段输出合同。
- `node_id`、`route_id`、`binding_id` 只有在当前节点一致性校验或结果回接确实需要时才可保留；它们不得参与创意、路由或质量结论推理。
- `input_id`、`asset_ref`、`output_ref` 只用于当前阶段真实素材或上游结果的精确选择和回接，不得据此猜测内容。
- `attempt` 只允许出现在其修复预算或重试语义确实依赖次数的 Gate/Repair 调用中；普通生成、理解、规划和 Rewriter 不接收该字段。
- Provider、model、真实 endpoint、adapter、凭据、账户和价格由 Runtime/Catalog 管理。Bound Rewriter 只接收已选 binding 的能力约束与 Prompt Profile，不接收供应商调用元数据。
- 复合内容包与分镜 Prompt 只接收 `slot_id`、期望输出类型和用途；child node、child Route 与 selector 拓扑始终保存在 Runtime，由 Runtime 在模型输出后重新附加。

## 使用边界

- HTML 只决定 Prompt 的身份、顺序、状态和分组；正文定义目标输入、输出与校验合同，Rust 证据用于说明当前实现状态，尚未落地的差距必须显式记录。
- 每个 Prompt 只处理本次调用交给它的输入和当前职责，不解释整条多模态链路，也不代替相邻 Prompt 工作。
- provider、model、endpoint、鉴权、价格和提交请求体由 Runtime、Catalog 与 adapter 管理；质量门只可返回 schema 已定义的抽象重试策略，不得点名或选择具体服务。
- 本目录是独立设计注册表，不表示已经由生产加载器部署；状态、调用位置、已知链路缺口及运行绑定统一记录在 manifest.json。
- ACP Image/Video Check 使用阶段专用 `acp_quality_check_input.v1`，但统一输出 `quality_report.v1`；这表示输入投影按模态最小化，不代表另建评分体系。
- ACP Image/Video Repair 只在 `quality_report.v1` 为可修复失败、`hard_fail=false` 且仍有一次预算时运行。Runtime 先在模型外校验 source report 的节点、路由与 provenance，再确定性投影 `quality_failure_slice.v1`；模型只接收 finding 相关的 `rewriter_context`、该只读失败切片和可解析证据，不接收完整报告、原 Provider Prompt 或完整原始 Rewriter 输入。Runtime 校验 correction context 后把它注入完整原输入，并让同一 Rewriter 重跑一次。JSON/schema 结构错误由 `STRICT_JSON_REPAIR` 处理。
- ACP Package 验收只比较当前节点目标、硬要求、内容包和子项质量报告，不重新加载完整用户请求或会话。
- `TEXT_TO_SPEECH_SPEC_REWRITER` 要求非空原文；发音表、暂停和强调只有在阶段输入真实提供且 Prompt Profile 明确支持时才可使用，声音引用授权仍由 Runtime 校验。
- 当前只有 Route Selector 使用 `{{ROUTE_CANDIDATES}}` 占位符；缺值、残留占位符或文件 hash 不匹配时必须拒绝加载。确定性 Prompt 是编译合同，不是占位符文本模板。

## Prompt 正文规范

- 正文按“职责、输入、处理、输出、字段说明、自检”组织；只写完成当前职责所需的信息。
- JSON 示例必须保持合法，并递归展开为一行一个字段；嵌套对象和数组对象同样换行，不在 JSON 内加入 `//` 注释。
- 字段解释放在 JSON 后，只说明含义、来源和关键约束，不重复完整系统背景。
- 图片相关 Prompt 应在现有 Schema 内覆盖主体细节、场景、构图与镜头、风格、色彩、光线、材质纹理、细节质感、文字策略和可观察验收标准。
- 视频相关 Prompt 在图片维度之外，还应覆盖时序、动作路径与速度、镜头运动、转场、跨帧连续性、稳定终帧和声音策略。
- 准确标题、字幕、数字、Logo 与 UI 文案采用程序化叠加；生成规格只保留可用文字安全区并禁止乱码、伪 UI 和随机标识。
- 目标合同可先于当前代码演进，但新增或变更字段必须写入 manifest 的运行差距和必要断言；缺少证据时使用合同允许的空值或明确说明未知。
- Prompt 正文不罗列模型不会接收的全局追踪字段；这类边界统一由本 README、manifest 和 Runtime assembler 保证。

## 目录与数量

| HTML 分组 | 范围 | 数量 |
|---|---:|---:|
| 路由与业务内容包 | 01–08 | 8 |
| 文档内容包与 DAG | 09–11 | 3 |
| 原子能力 Prompt | 12–19 | 8 |
| 理解与通用质量门 | 20–27 | 8 |
| ACP 内容验收与修复 | 28–32 | 5 |
| 严格 JSON 修复 | 33 | 1 |
| 确定性 fallback | 34–41 | 8 |
| 合计 | 01–41 | 41 |

HTML 状态合计：active 24 个、reserved 9 个、active_deterministic 8 个。

HTML 另将 `VIDEO_UNDERSTANDING_ANALYZER`、`VLM_IMAGE_UNDERSTANDING`、`VLM_VIDEO_UNDERSTANDING` 标为独立内容理解能力：它们保留各自 Prompt，但不进入多媒体内容产出链路。该范围标记记录在 manifest，不重复写入 Prompt 正文。

## 完整索引

| # | Prompt ID | 状态 | 文件 |
|---:|---|---|---|
| 01 | INTENT_CLASSIFIER | active | [INTENT_CLASSIFIER.md](路由与业务内容包/INTENT_CLASSIFIER.md) |
| 02 | ROUTE_ID_SELECTOR | active | [ROUTE_ID_SELECTOR.md](路由与业务内容包/ROUTE_ID_SELECTOR.md) |
| 03 | CONTENT_PACKAGE_GENERATOR | active | [CONTENT_PACKAGE_GENERATOR.md](路由与业务内容包/CONTENT_PACKAGE_GENERATOR.md) |
| 04 | VIDEO_STORYBOARD_GENERATOR | active | [VIDEO_STORYBOARD_GENERATOR.md](路由与业务内容包/VIDEO_STORYBOARD_GENERATOR.md) |
| 05 | TEXT_CONTENT_GENERATOR | active | [TEXT_CONTENT_GENERATOR.md](路由与业务内容包/TEXT_CONTENT_GENERATOR.md) |
| 06 | FACT_VERIFICATION_PLANNER | active | [FACT_VERIFICATION_PLANNER.md](路由与业务内容包/FACT_VERIFICATION_PLANNER.md) |
| 07 | VIDEO_UNDERSTANDING_ANALYZER | active | [VIDEO_UNDERSTANDING_ANALYZER.md](路由与业务内容包/VIDEO_UNDERSTANDING_ANALYZER.md) |
| 08 | WORD_DOCUMENT_PACKAGE_GENERATOR | active | [WORD_DOCUMENT_PACKAGE_GENERATOR.md](路由与业务内容包/WORD_DOCUMENT_PACKAGE_GENERATOR.md) |
| 09 | FIXED_LAYOUT_PDF_PACKAGE_GENERATOR | active | [FIXED_LAYOUT_PDF_PACKAGE_GENERATOR.md](文档内容包与%20DAG/FIXED_LAYOUT_PDF_PACKAGE_GENERATOR.md) |
| 10 | EXCEL_WORKBOOK_PACKAGE_GENERATOR | active | [EXCEL_WORKBOOK_PACKAGE_GENERATOR.md](文档内容包与%20DAG/EXCEL_WORKBOOK_PACKAGE_GENERATOR.md) |
| 11 | MULTIMODAL_DAG_PLANNER | active | [MULTIMODAL_DAG_PLANNER.md](文档内容包与%20DAG/MULTIMODAL_DAG_PLANNER.md) |
| 12 | IMAGE_PROMPT_REWRITER | active | [IMAGE_PROMPT_REWRITER.md](原子能力%20Prompt/IMAGE_PROMPT_REWRITER.md) |
| 13 | IMAGE_EDIT_PROMPT_REWRITER | active | [IMAGE_EDIT_PROMPT_REWRITER.md](原子能力%20Prompt/IMAGE_EDIT_PROMPT_REWRITER.md) |
| 14 | TEXT_TO_VIDEO_PROMPT_REWRITER | active | [TEXT_TO_VIDEO_PROMPT_REWRITER.md](原子能力%20Prompt/TEXT_TO_VIDEO_PROMPT_REWRITER.md) |
| 15 | IMAGE_TO_VIDEO_PROMPT_REWRITER | active | [IMAGE_TO_VIDEO_PROMPT_REWRITER.md](原子能力%20Prompt/IMAGE_TO_VIDEO_PROMPT_REWRITER.md) |
| 16 | VIDEO_EDIT_PROMPT_REWRITER | active | [VIDEO_EDIT_PROMPT_REWRITER.md](原子能力%20Prompt/VIDEO_EDIT_PROMPT_REWRITER.md) |
| 17 | TEXT_TO_SPEECH_SPEC_REWRITER | active | [TEXT_TO_SPEECH_SPEC_REWRITER.md](原子能力%20Prompt/TEXT_TO_SPEECH_SPEC_REWRITER.md) |
| 18 | AUDIO_TRANSCRIPTION_SPEC_REWRITER | active | [AUDIO_TRANSCRIPTION_SPEC_REWRITER.md](原子能力%20Prompt/AUDIO_TRANSCRIPTION_SPEC_REWRITER.md) |
| 19 | VLM_REFERENCE_UNDERSTANDING | active | [VLM_REFERENCE_UNDERSTANDING.md](原子能力%20Prompt/VLM_REFERENCE_UNDERSTANDING.md) |
| 20 | VLM_IMAGE_UNDERSTANDING | reserved | [VLM_IMAGE_UNDERSTANDING.md](理解与通用质量门/VLM_IMAGE_UNDERSTANDING.md) |
| 21 | VLM_VIDEO_UNDERSTANDING | reserved | [VLM_VIDEO_UNDERSTANDING.md](理解与通用质量门/VLM_VIDEO_UNDERSTANDING.md) |
| 22 | PROVIDER_PROMPT_SAFETY_CHECK | reserved | [PROVIDER_PROMPT_SAFETY_CHECK.md](理解与通用质量门/PROVIDER_PROMPT_SAFETY_CHECK.md) |
| 23 | TEXT_QUALITY_GATE | reserved | [TEXT_QUALITY_GATE.md](理解与通用质量门/TEXT_QUALITY_GATE.md) |
| 24 | PACKAGE_QUALITY_GATE | reserved | [PACKAGE_QUALITY_GATE.md](理解与通用质量门/PACKAGE_QUALITY_GATE.md) |
| 25 | VISUAL_QUALITY_GATE | reserved | [VISUAL_QUALITY_GATE.md](理解与通用质量门/VISUAL_QUALITY_GATE.md) |
| 26 | VIDEO_QUALITY_GATE | reserved | [VIDEO_QUALITY_GATE.md](理解与通用质量门/VIDEO_QUALITY_GATE.md) |
| 27 | AUDIO_QUALITY_GATE | reserved | [AUDIO_QUALITY_GATE.md](理解与通用质量门/AUDIO_QUALITY_GATE.md) |
| 28 | ACP_PACKAGE_QUALITY_CHECK | active | [ACP_PACKAGE_QUALITY_CHECK.md](ACP%20内容验收与修复/ACP_PACKAGE_QUALITY_CHECK.md) |
| 29 | ACP_IMAGE_QUALITY_CHECK | active | [ACP_IMAGE_QUALITY_CHECK.md](ACP%20内容验收与修复/ACP_IMAGE_QUALITY_CHECK.md) |
| 30 | ACP_IMAGE_QUALITY_REPAIR | active | [ACP_IMAGE_QUALITY_REPAIR.md](ACP%20内容验收与修复/ACP_IMAGE_QUALITY_REPAIR.md) |
| 31 | ACP_VIDEO_QUALITY_CHECK | active | [ACP_VIDEO_QUALITY_CHECK.md](ACP%20内容验收与修复/ACP_VIDEO_QUALITY_CHECK.md) |
| 32 | ACP_VIDEO_QUALITY_REPAIR | active | [ACP_VIDEO_QUALITY_REPAIR.md](ACP%20内容验收与修复/ACP_VIDEO_QUALITY_REPAIR.md) |
| 33 | STRICT_JSON_REPAIR | reserved | [STRICT_JSON_REPAIR.md](严格%20JSON%20修复/STRICT_JSON_REPAIR.md) |
| 34 | IMAGE_PROMPT_DETERMINISTIC_FALLBACK | active_deterministic | [IMAGE_PROMPT_DETERMINISTIC_FALLBACK.md](确定性%20fallback/IMAGE_PROMPT_DETERMINISTIC_FALLBACK.md) |
| 35 | IMAGE_EDIT_PROMPT_DETERMINISTIC_FALLBACK | active_deterministic | [IMAGE_EDIT_PROMPT_DETERMINISTIC_FALLBACK.md](确定性%20fallback/IMAGE_EDIT_PROMPT_DETERMINISTIC_FALLBACK.md) |
| 36 | TEXT_TO_VIDEO_PROMPT_DETERMINISTIC_FALLBACK | active_deterministic | [TEXT_TO_VIDEO_PROMPT_DETERMINISTIC_FALLBACK.md](确定性%20fallback/TEXT_TO_VIDEO_PROMPT_DETERMINISTIC_FALLBACK.md) |
| 37 | IMAGE_TO_VIDEO_PROMPT_DETERMINISTIC_FALLBACK | active_deterministic | [IMAGE_TO_VIDEO_PROMPT_DETERMINISTIC_FALLBACK.md](确定性%20fallback/IMAGE_TO_VIDEO_PROMPT_DETERMINISTIC_FALLBACK.md) |
| 38 | VIDEO_EDIT_PROMPT_DETERMINISTIC_FALLBACK | active_deterministic | [VIDEO_EDIT_PROMPT_DETERMINISTIC_FALLBACK.md](确定性%20fallback/VIDEO_EDIT_PROMPT_DETERMINISTIC_FALLBACK.md) |
| 39 | TEXT_TO_SPEECH_SPEC_DETERMINISTIC | active_deterministic | [TEXT_TO_SPEECH_SPEC_DETERMINISTIC.md](确定性%20fallback/TEXT_TO_SPEECH_SPEC_DETERMINISTIC.md) |
| 40 | AUDIO_TRANSCRIPTION_SPEC_DETERMINISTIC | active_deterministic | [AUDIO_TRANSCRIPTION_SPEC_DETERMINISTIC.md](确定性%20fallback/AUDIO_TRANSCRIPTION_SPEC_DETERMINISTIC.md) |
| 41 | VOICE_SEPARATION_SPEC_DETERMINISTIC | active_deterministic | [VOICE_SEPARATION_SPEC_DETERMINISTIC.md](确定性%20fallback/VOICE_SEPARATION_SPEC_DETERMINISTIC.md) |

manifest.json 是机器可读的唯一索引，记录文件 hash、输入输出键、运行时绑定和证据。文件正文与 manifest 不一致时必须拒绝加载，不允许静默降级。

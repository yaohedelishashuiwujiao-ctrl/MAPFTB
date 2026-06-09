# MAPFTB - Make a PPT for the Boss

面向职场汇报场景的证据驱动 AI PPT 工作台。用户导入文档、表格和零散笔记，系统通过可审计的 Agent 工作流生成汇报逻辑、逐页内容和可编辑 PPT，并标注每项结论的来源。

## 为什么做这个项目

普通 AI PPT 产品主要证明“调用了大模型”。本项目重点证明 AI 应用开发岗位更关心的能力：

- RAG：解析多种资料，混合检索并为结论提供来源引用
- Agent：通过可恢复工作流完成资料分析、提纲规划、内容生成和质量审查
- 结构化生成：使用 schema 约束大模型输出，稳定生成可编辑 PPT
- AI 评测：建立离线数据集，评估引用正确率、事实一致性和生成成功率
- 工程化：流式进度、任务重试、模型切换、成本与延迟观测、自动化测试
- 产品能力：解决“材料很多，但不知道如何向领导讲清楚”的真实问题

## 核心用户流程

1. 用户创建汇报任务，填写受众、目的、时长和表达风格。
2. 导入 PDF、DOCX、Markdown、CSV/XLSX 和图片。
3. 系统解析资料，抽取事实、指标、时间和来源位置。
4. Planner Agent 生成叙事提纲，用户确认或调整。
5. Writer Agent 生成逐页内容，Chart Agent 生成图表建议。
6. Reviewer Agent 检查无来源结论、数字冲突、重复页面和表达问题。
7. 用户查看每条结论的证据，编辑后导出 PPTX。

## MVP 范围

### 必须完成

- 支持 PDF、Markdown、CSV/XLSX 导入
- 文档切分、向量检索和关键词检索
- 可中断、可恢复的 Agent 工作流
- 提纲确认的人机协作节点
- 基于 Pydantic schema 的逐页结构化生成
- 结论级来源引用和证据查看
- 使用模板导出可编辑 PPTX
- 任务耗时、token 用量和模型费用统计
- 至少 30 个样例任务的离线评测集

### 暂不实现

- 通用模板商城
- 多人实时协作
- 自动生成复杂动画
- 自研或大规模微调模型
- 为了展示技术而堆叠 Multi-Agent

## 技术方案

### 推荐技术栈

- 前端：React + TypeScript
- 后端：FastAPI + Python 3.12 + Pydantic
- 工作流：LangGraph
- 数据库：PostgreSQL + pgvector；本地开发可先用 SQLite
- 文档解析：PyMuPDF、pandas、openpyxl
- PPT 导出：python-pptx
- 模型层：统一 ModelProvider，兼容 OpenAI-compatible API
- 可观测性：结构化日志 + OpenTelemetry；可选 Langfuse
- 测试：pytest + Vitest

### Agent 工作流

```text
ingest -> extract_facts -> retrieve_evidence -> plan_outline
       -> human_approve -> write_slides -> review_claims
       -> revise -> render_pptx
```

每个节点接收和返回明确的数据结构。工作流状态持久化，失败后从节点级检查点恢复。

### 关键数据结构

```text
SourceDocument
  id, filename, type, checksum

Evidence
  id, document_id, locator, text, metadata

Claim
  id, text, evidence_ids, confidence

Slide
  id, title, purpose, claims, visual_spec, speaker_notes

Presentation
  audience, objective, duration, outline, slides
```

## 评测设计

项目价值不能只靠截图展示，至少持续跟踪以下指标：

- 引用正确率：引用证据是否支持对应结论
- 数字一致率：生成内容中的数字是否与原资料一致
- 结论覆盖率：关键事实是否进入最终汇报
- PPTX 生成成功率：文件能否正常打开并编辑
- 人工修改率：生成后被用户修改的内容比例
- P50/P95 延迟、单任务 token 和费用

评测集使用公开年报、产品资料和人工构造的表格，避免把公司内部资料提交到仓库。

## 四周实施计划

### 第 1 周：纵向打通

- 搭建 FastAPI、React 和数据库
- 完成 Markdown/PDF 导入、证据检索和基础提纲生成
- 使用固定模板导出第一份 PPTX

### 第 2 周：可信生成

- 增加表格解析、混合检索和结论级引用
- 增加提纲确认节点与任务恢复
- 完成 Writer 和 Reviewer 工作流

### 第 3 周：评测与工程化

- 建立离线评测集和自动评测命令
- 增加 token、费用、延迟和失败原因观测
- 补齐单元测试、集成测试和 CI

### 第 4 周：作品化

- 优化交互和导出效果
- 准备 3 个端到端演示案例
- 编写架构文档、技术取舍、评测报告和演示视频
- 部署在线 Demo，并提供脱敏样例数据

## 简历表述模板

以下数字必须在项目完成后用真实评测结果替换：

> MAPFTB · 证据驱动的 AI 汇报生成工作台 ｜ 独立产品设计 / AI 全栈开发

- 基于 FastAPI、React 与 LangGraph 构建可恢复的 Agent 工作流，将多源资料处理为提纲、逐页内容和可编辑 PPTX，并支持人工确认与节点级失败重试。
- 设计混合检索与结论级引用机制，通过离线评测集验证引用正确率达到 `xx%`、数字一致率达到 `xx%`，降低无依据生成。
- 使用 Pydantic schema 约束结构化输出，建设模型适配、成本/延迟观测及自动化测试体系，实现 PPTX 生成成功率 `xx%`、P95 耗时 `xx s`。

## 面试演示脚本

1. 导入一份年报 PDF 和一份经营数据表格。
2. 输入：“给研发负责人做一份 8 页、10 分钟的季度复盘。”
3. 展示系统抽取出的事实与数字，以及提纲确认过程。
4. 展示某条结论对应的原始证据位置。
5. 故意制造数字冲突，展示 Reviewer 的发现和修订。
6. 导出 PPTX，打开并编辑其中的文字和图表。
7. 展示离线评测结果、调用成本和一次失败任务的恢复过程。

# MAPFTB - Make a PPT for the Boss

MAPFTB 是一个面向汽车竞品调研的 AI 情报与汇报系统。

长期目标是构建一个由采集 Agent 持续维护、包含多来源结构化数据与多模态证据的汽车竞品情报库，并将老板的一句话需求转换为可追溯、有数据、有分析、有图表的 PPT。

## 项目目标

```text
多来源定期采集
-> 车型版本与系统信息治理
-> 图片、视频与零部件分析
-> 调研 Agent 制定并执行计划
-> 受约束地生成分析报告与 PPT
```

项目重点解决：

- 汽车公开信息来源多、可信度不一
- 销售版本、公告型号和系统配置难以对齐
- 图片和拆解视频中的零部件信息难以检索
- 通用 AI 难以稳定完成专业分析和严格控制 PPT 输出
- 一次性调研结果无法持续沉淀和复用

## 当前阶段

项目处于基础架构设计阶段。第一阶段暂不实现 Agent、多模态和 PPT 生成，先建立可扩展的后端任务系统。

```text
FastAPI API
├── PostgreSQL：可靠业务数据与任务状态
├── Redis：缓存、任务 Broker 与短期状态
├── MinIO：PDF、图片、视频等对象存储
└── Celery Worker：执行耗时后台任务
```

计划中的基础技术栈：

- React + TypeScript
- FastAPI + Pydantic
- PostgreSQL + SQLAlchemy + Alembic
- Redis + Celery
- MinIO
- Docker Compose
- Prometheus + Grafana
- pytest + Locust + GitHub Actions

## 文档

- [项目总体方案](PROJECT_PLAN.md)
- [后端基础架构与学习指南](docs/BACKEND_FOUNDATION.md)
- [后端基础实践教程](docs/BACKEND_PRACTICAL_TUTORIAL.md)

## 开发原则

- 先以模块化单体建立清晰边界，再根据真实瓶颈拆分服务。
- PostgreSQL 保存可靠业务事实，Redis 不作为唯一事实来源。
- API 与耗时任务分离，所有后台任务需要支持幂等、重试和状态追踪。
- 数据、分析结论与 PPT 内容必须保留来源证据和不确定性。
- 每个开发阶段都需要可运行功能、自动化测试、指标与技术文档。

## 下一阶段

第一条可运行纵向链路：

```text
FastAPI 创建任务
-> PostgreSQL 保存任务
-> Redis 投递消息
-> Celery Worker 异步执行
-> PostgreSQL 更新最终状态
-> API 查询任务结果
```

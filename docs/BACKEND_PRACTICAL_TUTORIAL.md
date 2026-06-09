# MAPFTB 后端基础实践教程

这份教程通过一个最小但完整的“模拟调研任务”，解释 MAPFTB 基础后端中每项技术的作用和实践方式。

阅读目标不是复制代码，而是能够回答：

1. 一个请求如何从浏览器到达数据库？
2. 为什么耗时任务要交给 Worker？
3. PostgreSQL、Redis、MinIO 分别保存什么？
4. 如何处理任务重复执行、失败和重试？
5. 如何知道系统是否正常、是否扛得住并发？

本文中的代码是教学示例。正式开发时应拆分文件、补充异常处理和自动化测试。

## 1. 贯穿全文的业务案例

第一版暂时不接入爬虫和 AI，而是实现一个模拟调研任务：

```http
POST /api/v1/tasks
Content-Type: application/json

{
  "task_type": "demo_sleep",
  "duration_seconds": 5,
  "should_fail": false
}
```

API 不等待五秒，而是立即返回：

```http
HTTP/1.1 202 Accepted

{
  "id": "a9d2...",
  "status": "PENDING",
  "progress": 0
}
```

Celery Worker 在后台执行任务。客户端查询时可以看到：

```text
PENDING  ->  RUNNING  ->  SUCCEEDED
0%           20%-80%      100%
```

这个简单任务已经包含后续真实业务的基本形态：

| 模拟任务 | 未来真实任务 |
|---|---|
| 睡眠五秒 | 爬取汽车网站 |
| 更新进度 | 更新下载和解析进度 |
| 主动失败 | 网站超时、模型调用失败 |
| 返回简单结果 | 返回结构化汽车数据或 PPT |

## 2. 从外部观察整个系统

### 2.1 容器和进程图

```mermaid
flowchart LR
    User[浏览器或 curl]

    subgraph Host["开发机 / 服务器"]
        Nginx[Nginx 容器]
        API[FastAPI 容器<br/>API 进程]
        Worker[Celery Worker 容器<br/>后台进程]
        PG[(PostgreSQL 容器)]
        Redis[(Redis 容器)]
        MinIO[(MinIO 容器)]
    end

    User -->|HTTP| Nginx
    Nginx -->|/api| API
    API -->|SQL| PG
    API -->|发布任务| Redis
    Worker -->|领取任务| Redis
    Worker -->|更新状态| PG
    API -->|上传/下载| MinIO
    Worker -->|读写原始材料| MinIO
```

每个方框代表独立进程或容器。API 崩溃不应该导致数据库数据丢失；Worker 停止不应该阻止用户查询已有任务。

### 2.2 创建任务时序图

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant A as FastAPI
    participant P as PostgreSQL
    participant R as Redis
    participant W as Celery Worker

    U->>A: POST /api/v1/tasks
    A->>A: Pydantic 校验
    A->>P: INSERT task(PENDING)
    P-->>A: task_id
    A->>R: send_task(task_id)
    A-->>U: 202 + task_id
    W->>R: 获取消息
    W->>P: PENDING -> RUNNING
    loop 执行期间
        W->>P: 更新 progress
    end
    W->>P: RUNNING -> SUCCEEDED
    U->>A: GET /tasks/{id}
    A->>P: 查询任务
    A-->>U: 状态与结果
```

理解这张图后，后续所有技术都只是负责其中的一段。

## 3. Docker Compose：先让依赖可重复启动

### 3.1 Docker 解决什么问题

没有 Docker 时，每位开发者需要手动安装并配置 PostgreSQL、Redis 和 MinIO，版本和端口可能不同。

Docker Compose 将运行环境写成代码：

```text
compose.yaml
-> 启动哪些服务
-> 使用哪些镜像
-> 暴露哪些端口
-> 使用哪些环境变量
-> 数据保存到哪些 Volume
```

### 3.2 最小 Compose 示例

```yaml
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_DB: mapftb
      POSTGRES_USER: mapftb
      POSTGRES_PASSWORD: mapftb_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U mapftb -d mapftb"]
      interval: 5s
      timeout: 3s
      retries: 10

  redis:
    image: redis:8
    ports:
      - "6379:6379"
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: mapftb
      MINIO_ROOT_PASSWORD: mapftb_dev_password
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio_data:/data

volumes:
  postgres_data:
  redis_data:
  minio_data:
```

端口含义：

```text
PostgreSQL  5432  数据库连接
Redis       6379  Redis 客户端连接
MinIO       9000  S3 兼容 API
MinIO       9001  管理网页
```

### 3.3 容器网络如何理解

如果 FastAPI 在宿主机运行：

```text
PostgreSQL 地址：localhost:5432
Redis 地址：localhost:6379
```

如果 FastAPI 也在 Compose 容器中：

```text
PostgreSQL 地址：postgres:5432
Redis 地址：redis:6379
```

容器中的 `localhost` 指向容器自己，不是宿主机，也不是其他容器。

```mermaid
flowchart TB
    Host[宿主机 localhost]
    API[API 容器 localhost]
    PG[postgres:5432]

    Host -->|localhost:5432 映射端口| PG
    API -->|postgres:5432 Compose 服务名| PG
    API -. localhost:5432 指向 API 自己 .-> API
```

### 3.4 动手练习

```bash
docker compose up -d
docker compose ps
docker compose logs postgres
docker compose stop redis
docker compose start redis
```

观察：

- `stop` 后 Volume 是否仍然存在。
- Redis 停止时，未来的 readiness 检查应该如何变化。
- `docker compose down -v` 为什么是危险命令。

### 3.5 面试理解点

**`depends_on` 是否代表 PostgreSQL 已可用？**

不一定。它主要描述启动依赖。数据库进程启动后还需要初始化，因此应用仍需健康检查和连接重试。

## 4. 配置管理：代码不应该写死环境信息

### 4.1 为什么需要配置对象

下面的写法不可维护：

```python
# 错误示例
engine = create_engine("postgresql://mapftb:mapftb_dev@localhost:5432/mapftb")
```

它把密码、环境和代码绑在一起。应使用环境变量和统一配置对象。

### 4.2 Pydantic Settings 示例

```python
from functools import lru_cache

from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    app_name: str = "MAPFTB API"
    environment: str = "development"

    database_url: str
    redis_url: str
    minio_endpoint: str
    minio_access_key: str
    minio_secret_key: str

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

`.env.example` 只提供示例，不保存真实密码：

```dotenv
DATABASE_URL=postgresql+psycopg://mapftb:change_me@localhost:5432/mapftb
REDIS_URL=redis://localhost:6379/0
MINIO_ENDPOINT=localhost:9000
MINIO_ACCESS_KEY=mapftb
MINIO_SECRET_KEY=change_me
```

### 4.3 动手练习

1. 删除 `DATABASE_URL`，观察应用启动时的错误。
2. 将数据库端口改错，区分“配置校验失败”和“连接失败”。
3. 确认 `.env` 被 Git 忽略，`.env.example` 可以提交。

## 5. FastAPI：负责 HTTP 边界

### 5.1 最小应用

```python
from fastapi import FastAPI

app = FastAPI(title="MAPFTB API", version="0.1.0")


@app.get("/api/v1/health/live")
def live() -> dict[str, str]:
    return {"status": "ok"}
```

启动：

```bash
uvicorn app.main:app --reload
```

访问：

```text
http://localhost:8000/docs
http://localhost:8000/api/v1/health/live
```

### 5.2 一次 HTTP 请求包含什么

```http
POST /api/v1/tasks?notify=false HTTP/1.1
Host: localhost:8000
Content-Type: application/json
X-Request-ID: req-123

{"task_type":"demo_sleep","duration_seconds":5}
```

拆解：

```text
Method    POST                  想执行什么操作
Path      /api/v1/tasks         操作哪个资源
Query     notify=false          可选请求参数
Header    Content-Type          请求元信息
Body      JSON                  具体输入
```

### 5.3 `def` 和 `async def`

```python
@app.get("/sync")
def sync_endpoint():
    return {"mode": "sync"}


@app.get("/async")
async def async_endpoint():
    return {"mode": "async"}
```

重要规则：

```text
异步函数 + 异步 I/O 库       合理
异步函数 + 阻塞 I/O          会阻塞事件循环
异步函数 + 大量 CPU 计算      会阻塞事件循环
重型工作                     交给 Worker
```

错误示例：

```python
import time


@app.get("/bad")
async def bad_endpoint():
    time.sleep(10)  # 阻塞事件循环
    return {"done": True}
```

异步等待示例：

```python
import asyncio


@app.get("/better")
async def better_endpoint():
    await asyncio.sleep(10)
    return {"done": True}
```

但真实的十分钟调研任务即使可以异步等待，也不应留在 API 中，应交给 Worker。

### 5.4 Router 分组

```python
from fastapi import APIRouter

router = APIRouter(prefix="/api/v1/tasks", tags=["tasks"])


@router.get("")
def list_tasks():
    return []
```

在主应用中注册：

```python
app.include_router(router)
```

Router 应只处理 HTTP 语义，不直接编写复杂 SQL 或任务流程。

### 5.5 常用状态码

| 状态码 | 用途 |
|---:|---|
| `200 OK` | 查询或普通操作成功 |
| `201 Created` | 资源同步创建成功 |
| `202 Accepted` | 已接受异步任务，但尚未完成 |
| `204 No Content` | 操作成功且无需响应体 |
| `400 Bad Request` | 请求语义错误 |
| `404 Not Found` | 资源不存在 |
| `409 Conflict` | 状态冲突或重复资源 |
| `422 Unprocessable Entity` | 输入未通过 Schema 校验 |
| `503 Service Unavailable` | 依赖异常，服务暂不可用 |

### 5.6 动手练习

1. 增加 `/health/live`。
2. 增加 `/tasks/{task_id}`，先返回固定数据。
3. 在 `/docs` 中测试合法与非法 UUID。
4. 使用 `curl -i` 查看响应状态码和 Header。

## 6. Pydantic：把输入输出变成明确契约

### 6.1 为什么不能直接接收任意字典

如果请求直接接受 `dict`：

```python
def create_task(payload: dict):
    duration = payload["duration_seconds"]
```

系统不知道：

- 字段是否缺失。
- 类型是否正确。
- 持续时间是否合理。
- API 文档如何描述请求。

### 6.2 请求和响应 Schema

```python
from enum import StrEnum
from uuid import UUID

from pydantic import BaseModel, Field


class TaskStatus(StrEnum):
    PENDING = "PENDING"
    RUNNING = "RUNNING"
    RETRYING = "RETRYING"
    SUCCEEDED = "SUCCEEDED"
    FAILED = "FAILED"


class TaskCreate(BaseModel):
    task_type: str = Field(min_length=1, max_length=50)
    duration_seconds: int = Field(ge=1, le=30)
    should_fail: bool = False


class TaskRead(BaseModel):
    id: UUID
    task_type: str
    status: TaskStatus
    progress: int = Field(ge=0, le=100)
```

FastAPI 使用 Schema：

```python
@router.post("", response_model=TaskRead, status_code=202)
def create_task(payload: TaskCreate) -> TaskRead:
    ...
```

### 6.3 校验失败示例

请求：

```json
{
  "task_type": "demo_sleep",
  "duration_seconds": 999
}
```

Pydantic 会阻止请求进入业务逻辑，FastAPI 返回 `422`。这比任务执行到一半才发现参数非法更安全。

### 6.4 自定义校验示例

```python
from pydantic import BaseModel, model_validator


class TaskCreate(BaseModel):
    task_type: str
    duration_seconds: int
    should_fail: bool = False

    @model_validator(mode="after")
    def validate_demo_task(self):
        if self.task_type != "demo_sleep":
            raise ValueError("first version only supports demo_sleep")
        return self
```

### 6.5 动手练习

1. 将持续时间限制为 `1-30` 秒。
2. 尝试提交字符串 `"5"`，观察宽松校验行为。
3. 开启严格模式后再次尝试。
4. 给响应增加 `created_at`，观察 OpenAPI 文档变化。

## 7. PostgreSQL：保存可靠业务事实

### 7.1 为什么不能只依赖 Celery 状态

系统未来需要回答：

- 某任务是谁创建的？
- 输入是什么？
- 为什么失败？
- 重试过几次？
- 产生了哪些数据和文件？

这些属于业务记录，应长期保存在 PostgreSQL。

### 7.2 第一版任务表

```sql
CREATE TYPE task_status AS ENUM (
  'PENDING',
  'RUNNING',
  'RETRYING',
  'SUCCEEDED',
  'FAILED'
);

CREATE TABLE tasks (
  id UUID PRIMARY KEY,
  task_type VARCHAR(50) NOT NULL,
  status task_status NOT NULL,
  progress INTEGER NOT NULL DEFAULT 0,
  input_json JSONB NOT NULL,
  result_json JSONB,
  error_message TEXT,
  retry_count INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  started_at TIMESTAMPTZ,
  finished_at TIMESTAMPTZ,
  CHECK (progress BETWEEN 0 AND 100)
);

CREATE INDEX ix_tasks_status_created_at
ON tasks (status, created_at DESC);
```

### 7.3 约束为什么重要

应用代码可能有 Bug，但数据库约束是最后防线：

```text
PRIMARY KEY       防止重复任务 ID
NOT NULL          防止关键字段缺失
CHECK             防止进度变成 300
FOREIGN KEY       防止引用不存在的数据
UNIQUE            防止重复业务记录
```

### 7.4 事务示例

```sql
BEGIN;

INSERT INTO tasks (...);
INSERT INTO audit_logs (...);

COMMIT;
```

如果第二条 SQL 失败：

```sql
ROLLBACK;
```

两条写入都不会留下半成品。

### 7.5 原子领取任务

```sql
UPDATE tasks
SET status = 'RUNNING',
    started_at = now()
WHERE id = :task_id
  AND status IN ('PENDING', 'RETRYING')
RETURNING *;
```

并发的两个 Worker 同时执行时，只有一个能成功修改符合条件的行。

```mermaid
sequenceDiagram
    participant W1 as Worker 1
    participant DB as PostgreSQL
    participant W2 as Worker 2

    W1->>DB: UPDATE WHERE status=PENDING
    DB-->>W1: 更新 1 行，允许执行
    W2->>DB: UPDATE WHERE status=PENDING
    DB-->>W2: 更新 0 行，停止执行
```

### 7.6 索引如何理解

没有索引：

```text
查找所有 RUNNING 任务
-> 数据库可能扫描整张 tasks 表
```

有 `(status, created_at)` 索引：

```text
数据库先定位 RUNNING 区域
-> 再按 created_at 获取所需记录
```

使用执行计划验证：

```sql
EXPLAIN ANALYZE
SELECT *
FROM tasks
WHERE status = 'RUNNING'
ORDER BY created_at DESC
LIMIT 20;
```

不要看到字段就加索引。索引会增加写入成本和存储空间。

### 7.7 动手练习

1. 手工尝试插入 `progress=200`，观察约束错误。
2. 开启两个数据库会话，尝试并发更新同一任务。
3. 插入大量任务后，对比增加索引前后的执行计划。
4. 执行一次事务后主动回滚，确认数据未写入。

## 8. SQLAlchemy 与 Alembic：用代码管理数据库

### 8.1 ORM Model

```python
import uuid
from datetime import datetime, timezone
from enum import StrEnum

from sqlalchemy import CheckConstraint, DateTime, Enum, Integer, String
from sqlalchemy.dialects.postgresql import JSONB, UUID
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class TaskStatus(StrEnum):
    PENDING = "PENDING"
    RUNNING = "RUNNING"
    RETRYING = "RETRYING"
    SUCCEEDED = "SUCCEEDED"
    FAILED = "FAILED"


class Task(Base):
    __tablename__ = "tasks"
    __table_args__ = (
        CheckConstraint("progress BETWEEN 0 AND 100"),
    )

    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), primary_key=True, default=uuid.uuid4
    )
    task_type: Mapped[str] = mapped_column(String(50))
    status: Mapped[TaskStatus] = mapped_column(
        Enum(TaskStatus), default=TaskStatus.PENDING
    )
    progress: Mapped[int] = mapped_column(Integer, default=0)
    input_json: Mapped[dict] = mapped_column(JSONB)
    result_json: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )
```

### 8.2 Engine、Connection、Session

```mermaid
flowchart LR
    Service[Service]
    Session[Session<br/>一次工作单元]
    Pool[Engine 连接池]
    C1[数据库连接 1]
    C2[数据库连接 2]
    PG[(PostgreSQL)]

    Service --> Session
    Session --> Pool
    Pool --> C1
    Pool --> C2
    C1 --> PG
    C2 --> PG
```

- Engine：管理连接池和数据库方言。
- Connection：一条实际数据库连接。
- Session：跟踪 ORM 对象，并代表一次工作单元。

### 8.3 第一版同步还是异步 SQLAlchemy

两种方案都可行：

| 方案 | 优点 | 代价 |
|---|---|---|
| 同步 SQLAlchemy | FastAPI 和 Celery 可复用同一套数据访问代码，容易理解 | 高 I/O 并发时线程占用更多 |
| 异步 SQLAlchemy | API 等待数据库时可让出事件循环 | Celery 原生任务是同步模型，整合更复杂 |

对于第一条基础链路，建议先使用**同步 SQLAlchemy**，把事务、状态机和幂等做正确。FastAPI 的普通 `def` 路由会在线程池中执行。

后续压测证明数据库访问是瓶颈后，再评估 API 使用异步 SQLAlchemy。不要仅为了“异步”标签增加复杂度。

同步 Session 示例：

```python
from collections.abc import Generator

from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

engine = create_engine(
    settings.database_url,
    pool_pre_ping=True,
    pool_size=10,
    max_overflow=20,
)

SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)


def get_session() -> Generator[Session, None, None]:
    with SessionLocal() as session:
        yield session
```

### 8.4 `flush` 与 `commit`

```python
task = Task(...)
session.add(task)
session.flush()   # SQL 已发送，task.id 可用，但事务尚未提交
session.commit()  # 事务正式提交
```

如果后续失败：

```python
session.rollback()
```

Repository 不应随意 `commit`，否则 Service 无法控制完整事务。

### 8.5 Repository 示例

```python
from uuid import UUID

from sqlalchemy import select, update
from sqlalchemy.orm import Session


class TaskRepository:
    def __init__(self, session: Session):
        self.session = session

    def add(self, task: Task) -> Task:
        self.session.add(task)
        self.session.flush()
        return task

    def get(self, task_id: UUID) -> Task | None:
        return self.session.scalar(
            select(Task).where(Task.id == task_id)
        )

    def claim(self, task_id: UUID) -> bool:
        result = self.session.execute(
            update(Task)
            .where(
                Task.id == task_id,
                Task.status == TaskStatus.PENDING,
            )
            .values(status=TaskStatus.RUNNING)
        )
        return result.rowcount == 1
```

### 8.6 Alembic 迁移

```bash
alembic init migrations
alembic revision --autogenerate -m "create tasks table"
alembic upgrade head
alembic downgrade -1
alembic current
alembic history
```

迁移文件的意义：

```text
代码版本 A 对应数据库结构 A
代码版本 B 对应数据库结构 B
```

不要在生产环境依赖 `Base.metadata.create_all()` 自动改表，它不能可靠表达复杂升级和回滚过程。

### 8.7 动手练习

1. 创建 `tasks` Model 和迁移。
2. 升级后用数据库客户端查看表结构。
3. 新增 `error_message` 字段，再生成第二条迁移。
4. 执行降级，观察字段是否回滚。
5. 故意在事务中抛异常并验证回滚。

## 9. Service 与 Repository：把业务规则放对位置

### 9.1 分层调用关系

```mermaid
flowchart LR
    Router[Router<br/>HTTP 语义]
    Service[Service<br/>业务规则与事务]
    Repo[Repository<br/>数据库操作]
    DB[(PostgreSQL)]

    Router --> Service
    Service --> Repo
    Repo --> DB
```

### 9.2 Service 示例

```python
class TaskService:
    def __init__(self, session: Session):
        self.session = session
        self.repository = TaskRepository(session)

    def create(self, payload: TaskCreate) -> Task:
        task = Task(
            task_type=payload.task_type,
            status=TaskStatus.PENDING,
            progress=0,
            input_json=payload.model_dump(),
        )
        self.repository.add(task)
        self.session.commit()
        return task

    def mark_succeeded(self, task_id: UUID, result: dict) -> Task:
        task = self.repository.get(task_id)
        if task is None:
            raise TaskNotFound(task_id)
        if task.status != TaskStatus.RUNNING:
            raise InvalidTaskTransition(task.status, TaskStatus.SUCCEEDED)

        task.status = TaskStatus.SUCCEEDED
        task.progress = 100
        task.result_json = result
        self.session.commit()
        return task
```

### 9.3 状态转换规则

```python
ALLOWED_TRANSITIONS = {
    TaskStatus.PENDING: {TaskStatus.RUNNING, TaskStatus.FAILED},
    TaskStatus.RUNNING: {
        TaskStatus.RETRYING,
        TaskStatus.SUCCEEDED,
        TaskStatus.FAILED,
    },
    TaskStatus.RETRYING: {TaskStatus.RUNNING, TaskStatus.FAILED},
    TaskStatus.SUCCEEDED: set(),
    TaskStatus.FAILED: set(),
}
```

这样可以阻止：

```text
SUCCEEDED -> RUNNING
FAILED -> SUCCEEDED
```

如果未来需要手动重试，应创建明确的重试状态或新任务记录，而不是随意修改历史状态。

### 9.4 动手练习

1. 写一个不接数据库的状态转换单元测试。
2. 尝试将成功任务重新改为运行中，确认抛出异常。
3. 思考“取消任务”需要增加哪些状态。

## 10. Redis：快速但不是最终事实来源

### 10.1 Redis 在第一版中的三个角色

```mermaid
flowchart TB
    Redis[(Redis)]
    Broker[Celery Broker<br/>传递任务消息]
    Progress[短期进度<br/>快速读取]
    Cache[热点查询缓存<br/>后续使用]

    Redis --> Broker
    Redis --> Progress
    Redis --> Cache
```

第一版最重要的是 Broker。缓存可以后续再做，避免为了使用 Redis 而缓存所有接口。

### 10.2 常用数据结构示例

```text
String:
  task:a9d2:progress -> "60"

Hash:
  task:a9d2 -> {status: RUNNING, progress: 60}

Set:
  active_task_ids -> {a9d2, b8c1}

Sorted Set:
  task_schedule -> {task_id: timestamp}
```

Python 示例：

```python
from redis import Redis

redis = Redis.from_url(settings.redis_url, decode_responses=True)

redis.hset(
    "task:a9d2",
    mapping={"status": "RUNNING", "progress": 60},
)
redis.expire("task:a9d2", 3600)

progress = redis.hget("task:a9d2", "progress")
```

### 10.3 TTL 的意义

Redis 中的短期任务进度无需永久保存：

```text
任务执行期间：频繁更新 Redis，读取快
任务完成以后：最终结果写 PostgreSQL
一小时后：Redis 进度自动过期
```

### 10.4 Cache Aside 示例

```python
def get_task_summary(task_id: str) -> dict:
    key = f"task-summary:{task_id}"

    cached = redis.get(key)
    if cached:
        return json.loads(cached)

    summary = load_summary_from_postgres(task_id)
    redis.set(key, json.dumps(summary), ex=60)
    return summary
```

更新时：

```python
update_task_in_postgres(task_id)
redis.delete(f"task-summary:{task_id}")
```

### 10.5 不要这样使用 Redis

```text
错误：任务最终结果只存在 Redis
错误：没有 TTL 地缓存所有查询
错误：使用分布式锁替代数据库唯一约束
错误：将巨大视频或文档塞进 Redis
```

### 10.6 动手练习

```bash
redis-cli
PING
SET demo hello EX 10
GET demo
TTL demo
HSET task:1 status RUNNING progress 20
HGETALL task:1
```

观察 Key 过期后，为什么 PostgreSQL 中的任务仍然必须存在。

## 11. Celery：将耗时工作移出 HTTP 请求

### 11.1 Celery 的角色

```mermaid
flowchart LR
    API[FastAPI Producer]
    Broker[(Redis Broker)]
    W1[Celery Worker 1]
    W2[Celery Worker 2]
    DB[(PostgreSQL)]

    API -->|发布 task_id| Broker
    Broker -->|分发| W1
    Broker -->|分发| W2
    W1 -->|更新状态| DB
    W2 -->|更新状态| DB
```

Celery 消息中只传递稳定、简单的数据，例如 `task_id`：

```python
run_demo_task.delay(str(task.id))
```

不要传递 ORM 对象：

```python
# 错误：ORM 对象难以序列化，而且消息中的对象很快过期
run_demo_task.delay(task)
```

### 11.2 Celery 应用配置

```python
from celery import Celery

celery_app = Celery(
    "mapftb",
    broker=settings.redis_url,
)

celery_app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    task_acks_late=True,
    worker_prefetch_multiplier=1,
    task_track_started=True,
)
```

配置理解：

```text
task_acks_late:
  Worker 执行后再确认消息，崩溃时消息可以重新投递。

worker_prefetch_multiplier=1:
  避免某个 Worker 一次预取过多长任务。

JSON serializer:
  避免不安全且难以跨语言的序列化方式。
```

### 11.3 任务示例

```python
import time
from uuid import UUID


@celery_app.task(
    bind=True,
    autoretry_for=(TemporaryTaskError,),
    retry_backoff=True,
    retry_kwargs={"max_retries": 3},
)
def run_demo_task(self, task_id: str) -> None:
    with SessionLocal() as session:
        service = TaskService(session)

        if not service.claim(UUID(task_id)):
            return

        task = service.get(UUID(task_id))
        duration = task.input_json["duration_seconds"]

        try:
            for second in range(duration):
                time.sleep(1)
                progress = int((second + 1) / duration * 100)
                service.update_progress(task.id, progress)

            if task.input_json["should_fail"]:
                raise PermanentTaskError("requested demo failure")

            service.mark_succeeded(task.id, {"message": "demo complete"})
        except TemporaryTaskError:
            service.mark_retrying(task.id)
            raise
        except Exception as exc:
            service.mark_failed(task.id, str(exc))
            raise
```

教学代码中省略了部分异常处理。实际实现必须保证异常发生时 Session 正确回滚。

### 11.4 至少一次投递与幂等

```text
任务可能不执行：不可接受，需要可靠投递和修复机制
任务可能重复执行：正常现象，业务必须幂等
```

第一版幂等保护：

```python
def claim(self, task_id: UUID) -> bool:
    # 只有 PENDING 才能变成 RUNNING
    # 重复消息无法再次成功领取
    ...
```

### 11.5 重试分类

```python
class TemporaryTaskError(Exception):
    """网络超时、429 等稍后可能恢复的问题。"""


class PermanentTaskError(Exception):
    """非法文件、无法解析等重试也不会恢复的问题。"""
```

不要对所有 `Exception` 自动重试。代码 Bug 无限重试只会持续污染队列。

### 11.6 队列隔离

未来不同任务应使用不同队列：

```text
crawl     网络爬取，I/O 密集
video     FFmpeg，CPU 密集
ai        模型调用，受 API 限流影响
ppt       PPT 文件生成
```

```mermaid
flowchart LR
    Broker[(Redis)]
    Crawl[crawl queue]
    Video[video queue]
    AI[ai queue]
    CW[Crawler Workers]
    VW[Video Workers]
    AW[AI Workers]

    Broker --> Crawl --> CW
    Broker --> Video --> VW
    Broker --> AI --> AW
```

这样视频任务不会占满全部 Worker，导致简单爬虫也无法执行。

### 11.7 动手练习

1. Worker 停止时创建任务，确认状态保持 `PENDING`。
2. 启动 Worker，观察任务开始执行。
3. 执行期间强制停止 Worker，再启动并观察任务行为。
4. 向 Broker 重复发送同一 `task_id`，验证幂等保护。
5. 创建一个临时失败和一个永久失败案例，观察重试差异。

## 12. MinIO：保存大文件和原始证据

### 12.1 为什么使用对象存储

汽车情报系统未来会保存：

- 网页原始快照
- PDF 配置表
- 汽车图片
- 拆解视频
- 视频关键帧
- 最终 PPTX

对象存储使用 Bucket 和 Object Key 定位文件：

```text
Bucket: raw-sources
Object Key: autohome/series-7067/2026-06-10/page.html
```

### 12.2 数据库与 MinIO 的关系

```mermaid
flowchart LR
    File[汽车图片二进制]
    MinIO[(MinIO<br/>保存内容)]
    Meta[PostgreSQL<br/>保存元数据]

    File --> MinIO
    MinIO -->|object_key| Meta
```

数据库记录示例：

```text
id
object_key
content_hash
content_type
size_bytes
source_url
fetched_at
```

### 12.3 上传示例

```python
from io import BytesIO

from minio import Minio

client = Minio(
    settings.minio_endpoint,
    access_key=settings.minio_access_key,
    secret_key=settings.minio_secret_key,
    secure=False,
)

content = b"example raw page"

client.put_object(
    bucket_name="raw-sources",
    object_name="demo/page.html",
    data=BytesIO(content),
    length=len(content),
    content_type="text/html",
)
```

### 12.4 哈希去重

```python
import hashlib


def sha256_bytes(content: bytes) -> str:
    return hashlib.sha256(content).hexdigest()
```

处理流程：

```text
上传时计算 SHA-256
-> 查询数据库是否已有相同哈希
-> 已存在则复用对象
-> 不存在则上传并记录元数据
```

哈希相同说明内容相同，但不同来源仍可分别保存“来源引用关系”。

### 12.5 Presigned URL

私有文件不应永久公开。API 可以生成短期下载地址：

```python
from datetime import timedelta

url = client.presigned_get_object(
    "raw-sources",
    "demo/page.html",
    expires=timedelta(minutes=10),
)
```

前端在十分钟内可以下载，地址过期后失效。

### 12.6 动手练习

1. 创建 `raw-sources` Bucket。
2. 上传一个文本文件并从管理页面查看。
3. 重复上传同一内容并通过哈希识别重复。
4. 创建十分钟 Presigned URL。
5. 思考删除数据库记录时是否应该立即删除对象。

## 13. 健康检查：活着不等于能提供服务

### 13.1 Liveness 与 Readiness

```mermaid
flowchart LR
    Live[/health/live/]
    Ready[/health/ready/]
    Process[API 进程]
    PG[(PostgreSQL)]
    Redis[(Redis)]
    MinIO[(MinIO)]

    Live --> Process
    Ready --> PG
    Ready --> Redis
    Ready --> MinIO
```

```text
Liveness:
  进程是否还活着。
  失败时可以考虑重启进程。

Readiness:
  当前是否具备接收请求的条件。
  失败时应停止向该实例转发流量。
```

### 13.2 Readiness 示例

```python
from fastapi import APIRouter, Response, status
from sqlalchemy import text


@router.get("/ready")
def ready(response: Response) -> dict:
    checks = {
        "postgres": check_postgres(),
        "redis": check_redis(),
        "minio": check_minio(),
    }

    healthy = all(item["ok"] for item in checks.values())
    if not healthy:
        response.status_code = status.HTTP_503_SERVICE_UNAVAILABLE

    return {"status": "ok" if healthy else "unavailable", "checks": checks}
```

健康检查必须设置短超时，不能因为依赖故障让检查接口卡住几十秒。

### 13.3 动手练习

1. 正常访问 readiness。
2. 停止 Redis，确认返回 `503`。
3. 确认 liveness 仍返回 `200`。
4. 恢复 Redis，确认无需重启 API。

## 14. Nginx：统一入口和反向代理

### 14.1 为什么 API 前面需要 Nginx

```mermaid
flowchart LR
    User[用户]
    Nginx[Nginx<br/>统一入口]
    Static[React 静态文件]
    API1[FastAPI 实例 1]
    API2[FastAPI 实例 2]

    User --> Nginx
    Nginx --> Static
    Nginx --> API1
    Nginx --> API2
```

Nginx 可以负责：

- TLS 终止
- 静态文件服务
- API 反向代理
- 请求体大小限制
- 基础限流
- 多 API 实例负载均衡

### 14.2 最小配置

```nginx
upstream mapftb_api {
    server api:8000;
}

server {
    listen 80;

    location /api/ {
        proxy_pass http://mapftb_api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Request-ID $request_id;
    }
}
```

### 14.3 SSE 注意事项

SSE 需要及时把事件推给客户端。Nginx 默认缓冲可能导致前端迟迟看不到进度：

```nginx
location /api/v1/task-events/ {
    proxy_pass http://mapftb_api;
    proxy_buffering off;
    proxy_cache off;
    proxy_read_timeout 3600s;
}
```

### 14.4 动手练习

1. 通过 Nginx 访问 `/api/v1/health/live`。
2. 停止 API，观察 Nginx 返回什么错误。
3. 查看 `X-Request-ID` 是否传递到 API。

## 15. 日志与请求 ID：让故障可以追踪

### 15.1 普通文本日志的问题

```text
task failed
```

无法知道哪个请求、哪个任务、为什么失败。

结构化日志：

```json
{
  "timestamp": "2026-06-10T10:00:00Z",
  "level": "ERROR",
  "event": "task_failed",
  "request_id": "req-123",
  "task_id": "a9d2",
  "error_type": "TemporaryTaskError",
  "retry_count": 2
}
```

### 15.2 Request ID 流程

```mermaid
sequenceDiagram
    participant U as User
    participant N as Nginx
    participant A as FastAPI
    participant W as Worker

    U->>N: HTTP 请求
    N->>A: X-Request-ID=req-123
    A->>A: 日志记录 req-123
    A->>W: task_id + origin_request_id
    W->>W: 日志记录 task_id 与 req-123
```

这样可以从一次用户请求追踪到后台任务。

### 15.3 日志原则

应该记录：

- 关键状态转换
- 外部依赖调用结果
- 重试和失败原因
- 请求 ID、任务 ID、耗时

不应该记录：

- 密码和 Token
- 完整敏感资料
- 每次循环的大量无价值日志

### 15.4 动手练习

1. 为每个请求生成或接收 `X-Request-ID`。
2. 创建任务时将 Request ID 写入任务上下文。
3. 模拟失败并根据 Request ID 找到完整日志。

## 16. Prometheus 与 Grafana：从感觉变成指标

### 16.1 三类核心指标

| 类型 | 含义 | 示例 |
|---|---|---|
| Counter | 只增加的累计值 | 请求总数、失败任务总数 |
| Gauge | 可增可减的当前值 | 运行中任务数、队列长度 |
| Histogram | 数值分布 | API 延迟、任务执行时间 |

### 16.2 指标示例

```python
from prometheus_client import Counter, Gauge, Histogram

HTTP_REQUESTS = Counter(
    "mapftb_http_requests_total",
    "Total HTTP requests",
    ["method", "route", "status"],
)

RUNNING_TASKS = Gauge(
    "mapftb_running_tasks",
    "Current running tasks",
)

TASK_DURATION = Histogram(
    "mapftb_task_duration_seconds",
    "Task execution duration",
    ["task_type"],
)
```

标签必须控制基数：

```text
合理标签：method、route、status、task_type
危险标签：user_id、task_id、完整 URL
```

如果每个任务 ID 都成为标签，Prometheus 会产生海量时间序列。

### 16.3 RED 方法

```text
Rate      每秒请求数
Errors    错误率
Duration  请求延迟
```

对于 Worker 额外观察：

```text
队列长度
排队时间
执行时间
成功率
重试率
```

### 16.4 动手练习

1. 增加 HTTP 请求计数。
2. 增加任务执行耗时 Histogram。
3. 连续创建十个任务，观察指标变化。
4. 比较平均值和 P95 为什么不同。

## 17. pytest：固定系统行为

### 17.1 测试金字塔

```mermaid
flowchart TB
    E2E[少量端到端测试<br/>完整系统]
    Integration[适量集成测试<br/>API + 数据库]
    Unit[大量单元测试<br/>业务规则]

    E2E --> Integration --> Unit
```

### 17.2 状态机单元测试

```python
import pytest


def test_succeeded_task_cannot_return_to_running():
    with pytest.raises(InvalidTaskTransition):
        validate_transition(
            current=TaskStatus.SUCCEEDED,
            target=TaskStatus.RUNNING,
        )
```

这种测试快、稳定，并能精确说明业务规则。

### 17.3 API 测试

```python
from fastapi.testclient import TestClient


def test_create_task(client: TestClient):
    response = client.post(
        "/api/v1/tasks",
        json={
            "task_type": "demo_sleep",
            "duration_seconds": 3,
            "should_fail": False,
        },
    )

    assert response.status_code == 202
    assert response.json()["status"] == "PENDING"
```

### 17.4 测试外部依赖

不同测试采用不同策略：

```text
单元测试：
  Mock Repository、Redis 或 MinIO，快速验证业务规则。

集成测试：
  使用真实 PostgreSQL、Redis，验证 SQL 和事务行为。

端到端测试：
  启动 API 与 Worker，验证完整任务状态变化。
```

不要所有测试都 Mock，否则无法发现迁移、SQL 和连接配置问题。

### 17.5 Celery 测试

可以分层：

1. 把实际业务逻辑写成普通函数或 Service，直接测试。
2. 测试 Celery Task 是否正确调用 Service。
3. 少量端到端测试使用真实 Worker。

Celery 的 eager 模式可以帮助测试，但它不能完全复现真实 Broker 和 Worker 行为。

### 17.6 动手练习

至少编写：

```text
状态转换单元测试
任务创建 API 集成测试
非法输入测试
任务不存在测试
Worker 成功执行测试
Worker 永久失败测试
重复任务幂等测试
```

## 18. Locust：验证并发设计，不是制造漂亮数字

### 18.1 三个容易混淆的指标

```text
并发用户数：
  同一时间有多少模拟用户。

QPS / RPS：
  每秒完成多少请求。

响应时间：
  单个请求需要多久，重点关注 P95 和 P99。
```

### 18.2 最小压测脚本

```python
from locust import HttpUser, between, task


class MAPFTBUser(HttpUser):
    wait_time = between(0.5, 2)

    @task(3)
    def list_tasks(self):
        self.client.get("/api/v1/tasks")

    @task(1)
    def create_task(self):
        self.client.post(
            "/api/v1/tasks",
            json={
                "task_type": "demo_sleep",
                "duration_seconds": 3,
                "should_fail": False,
            },
        )
```

### 18.3 压测观察顺序

```mermaid
flowchart LR
    Load[增加并发]
    Metrics[观察 QPS / P95 / 错误率]
    API[检查 API CPU / 内存]
    DB[检查 SQL / 连接池]
    Queue[检查队列长度]
    Fix[一次只优化一个问题]

    Load --> Metrics --> API --> DB --> Queue --> Fix
```

不要只报告“支持 1000 并发”。应说明：

```text
测试环境
请求组合
持续时间
P95
错误率
数据库和 Worker 配置
发现的瓶颈
优化前后变化
```

### 18.4 动手练习

1. 只压测 liveness，建立基准。
2. 压测任务查询，观察数据库读取。
3. 压测任务创建，观察数据库写入和队列长度。
4. 停止 Worker 后继续创建任务，观察 Broker 堆积。
5. 恢复 Worker，观察消化速度。

## 19. GitHub Actions：让质量检查自动执行

### 19.1 CI 流程

```mermaid
flowchart LR
    Push[Push / Pull Request]
    Lint[Ruff / mypy]
    Test[pytest]
    Build[构建镜像]
    Result[允许或阻止合并]

    Push --> Lint --> Test --> Build --> Result
```

### 19.2 最小工作流示例

```yaml
name: backend-ci

on:
  push:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_DB: mapftb_test
          POSTGRES_USER: mapftb
          POSTGRES_PASSWORD: mapftb_test
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -e ".[dev]"
      - run: ruff check .
      - run: pytest
```

### 19.3 动手练习

1. 提交一个会导致 Ruff 失败的问题。
2. 观察 CI 阻止通过。
3. 修复后再次推送。
4. 思考迁移检查如何加入 CI。

## 20. 从创建任务到完成任务的伪代码总览

### 20.1 Router

```python
@router.post("", response_model=TaskRead, status_code=202)
def create_task(
    payload: TaskCreate,
    session: Session = Depends(get_session),
) -> TaskRead:
    service = TaskService(session)
    task = service.create(payload)

    try:
        run_demo_task.delay(str(task.id))
    except Exception as exc:
        service.mark_dispatch_failed(task.id, str(exc))
        raise ServiceUnavailableError("task broker unavailable") from exc

    return TaskRead.model_validate(task)
```

### 20.2 Service

```python
def create(self, payload: TaskCreate) -> Task:
    task = Task(
        task_type=payload.task_type,
        input_json=payload.model_dump(),
        status=TaskStatus.PENDING,
    )
    self.repository.add(task)
    self.session.commit()
    return task
```

### 20.3 Worker

```python
@celery_app.task(bind=True)
def run_demo_task(self, task_id: str):
    with SessionLocal() as session:
        service = TaskService(session)

        if not service.claim(UUID(task_id)):
            return

        try:
            execute_demo(service, UUID(task_id))
        except TemporaryTaskError as exc:
            service.mark_retrying(UUID(task_id), str(exc))
            raise self.retry(exc=exc, countdown=5)
        except Exception as exc:
            service.mark_failed(UUID(task_id), str(exc))
            raise
```

### 20.4 当前实现的可靠性缺口

上面的简单实现仍存在：

```text
数据库创建任务成功
-> Broker 消息发送失败
-> 任务可能停留或被标记失败
```

未来使用 Transactional Outbox 解决：

```mermaid
flowchart LR
    API[API]
    Tx[同一数据库事务]
    Task[(tasks)]
    Outbox[(outbox_events)]
    Publisher[事件发布器]
    Redis[(Redis Broker)]

    API --> Tx
    Tx --> Task
    Tx --> Outbox
    Publisher --> Outbox
    Publisher --> Redis
```

第一版不必立刻实现 Outbox，但必须知道当前系统的边界。

## 21. 一天实践路线

### 上午：让请求进入数据库

```text
1. 启动 PostgreSQL、Redis、MinIO
2. 创建 FastAPI 应用和 liveness
3. 创建 Pydantic Schema
4. 创建 SQLAlchemy Task Model
5. 创建 Alembic migration
6. 实现创建和查询任务
```

验收：

```text
POST /tasks 返回 202
GET /tasks/{id} 能查到 PENDING
非法输入返回 422
```

### 下午：让 Worker 执行任务

```text
1. 配置 Celery
2. Worker 根据 task_id 领取任务
3. 实现状态机和进度更新
4. 实现成功与失败路径
5. 编写幂等测试
6. 实现 readiness
```

验收：

```text
任务经历 PENDING -> RUNNING -> SUCCEEDED
失败任务进入 FAILED
重复消息不会重复执行
停止 Redis 后 readiness 返回 503
```

### 晚上：补测试和记录

```text
1. pytest 覆盖核心路径
2. 运行一次简单 Locust 压测
3. 记录遇到的问题
4. 更新 README
5. 提交 Git
```

## 22. 学完后的自检问题

如果能够不看文档回答这些问题，说明已经掌握基础框架：

1. 为什么 `POST /tasks` 返回 `202` 而不是 `200`？
2. 为什么不能只使用 Redis 保存任务状态？
3. Worker 为什么可能重复执行同一个任务？
4. 如何使用数据库状态条件保证幂等领取？
5. `flush`、`commit` 和 `rollback` 有什么区别？
6. 为什么 Repository 不应该自行决定所有事务提交？
7. 为什么大文件放 MinIO，元数据放 PostgreSQL？
8. Liveness 与 Readiness 有什么区别？
9. `async def` 中调用 `time.sleep` 会发生什么？
10. 如何区分应该重试与不应该重试的错误？
11. 为什么 Prometheus 标签不能使用 `task_id`？
12. 如何通过压测判断瓶颈在 API、数据库还是 Worker？

## 23. 推荐实践原则

- 每增加一项技术，先说明它解决的具体问题。
- 每个可靠性设计都必须有对应失败场景。
- 先让业务状态正确，再优化并发性能。
- 先用 PostgreSQL 约束和事务保证正确性，再考虑 Redis 锁。
- 任务默认按照可能重复执行设计。
- 不要在 API 请求中执行不可控的耗时工作。
- 不要用复杂基础设施掩盖不清晰的业务模型。
- 用测试、日志和指标证明系统行为，而不是依靠感觉。

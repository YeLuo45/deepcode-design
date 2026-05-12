# Session Management

> DeepCode 持久化会话与状态管理架构设计

## 1. Overview

DeepCode 采用持久化会话机制，所有交互历史存储在 JSONL 文件中，支持跨会话恢复和日志追踪。

```
~/.deepcode/sessions/
├── <session_id>/
│   ├── session.jsonl       # 完整会话历史
│   ├── planning_meta.json  # 规划元数据
│   └── deepcode_lab/
│       └── tasks/
│           └── <task_id>/
│               ├── logs/
│               │   ├── system.jsonl
│               │   ├── llm.jsonl
│               │   └── mcp.jsonl
│               └── workspace/
│                   └── (generated files)
```

---

## 2. Session Storage

### 2.1 Session Directory Structure

| Path | Type | Description |
|------|------|-------------|
| `sessions/` | Directory | 根会话目录 |
| `sessions/<id>/` | Directory | 单个会话目录 |
| `session.jsonl` | File | 完整会话历史 |
| `planning_meta.json` | File | 规划阶段元数据 |
| `deepcode_lab/` | Directory | 任务工作区根目录 |
| `uploads/` | Directory | 上传的源文件（共享） |

### 2.2 Session JSONL Format

```json
{"type": "user", "content": "...", "timestamp": "2026-05-12T10:00:00Z", "id": "msg_001"}
{"type": "assistant", "content": "...", "timestamp": "2026-05-12T10:00:01Z", "id": "msg_002"}
{"type": "tool_call", "tool": "read_file", "args": {...}, "timestamp": "2026-05-12T10:00:02Z", "id": "msg_003"}
{"type": "tool_result", "tool": "read_file", "result": "...", "timestamp": "2026-05-12T10:00:03Z", "id": "msg_004"}
```

---

## 3. Session Models

### 3.1 Session Model

```python
@dataclass
class Session:
    """会话模型"""
    id: str                           # Session ID (UUID)
    title: str                        # 会话标题
    created_at: datetime              # 创建时间
    updated_at: datetime              # 更新时间
    status: SessionStatus             # 会话状态
    task_ids: list[str]               # 关联的 Task ID
    model: str                        # 使用的模型
    metadata: dict[str, Any]          # 额外元数据
```

### 3.2 SessionStatus Enum

```python
class SessionStatus(Enum):
    """会话状态"""
    ACTIVE = "active"                 # 活跃会话
    IDLE = "idle"                    # 空闲会话
    COMPLETED = "completed"           # 已完成
    ABANDONED = "abandoned"          # 已废弃
```

### 3.3 Task Model

```python
@dataclass
class Task:
    """任务模型"""
    id: str                           # Task ID
    session_id: str                   # 父会话 ID
    paper_path: str                   # 源论文路径
    status: TaskStatus                # 任务状态
    created_at: datetime              # 创建时间
    updated_at: datetime              # 更新时间
    completed_at: datetime | None     # 完成时间
    error: str | None                # 错误信息
```

### 3.4 TaskStatus Enum

```python
class TaskStatus(Enum):
    """任务状态"""
    PENDING = "pending"              # 等待处理
    RUNNING = "running"              # 运行中
    WAITING_FOR_INPUT = "waiting_for_input"  # 等待输入
    COMPLETED = "completed"          # 已完成
    ABORTED = "aborted"              # 已中止
    INTERRUPTED = "interrupted"      # 被中断
    FAILED = "failed"                # 失败
```

---

## 4. Session Store

### 4.1 SessionStore Interface

```python
class SessionStore:
    """会话存储接口"""
    
    def __init__(self, base_dir: Path):
        self.base_dir = base_dir
        self.sessions_dir = base_dir / "sessions"
        self.sessions_dir.mkdir(parents=True, exist_ok=True)
    
    async def create_session(
        self,
        title: str | None = None,
        model: str = "gpt-4o"
    ) -> Session:
        """创建新会话"""
        ...
    
    async def get_session(self, session_id: str) -> Session | None:
        """获取会话"""
        ...
    
    async def list_sessions(
        self,
        status: SessionStatus | None = None,
        limit: int = 50
    ) -> list[Session]:
        """列出会话"""
        ...
    
    async def update_session(
        self,
        session_id: str,
        **kwargs
    ) -> Session:
        """更新会话"""
        ...
    
    async def delete_session(self, session_id: str) -> bool:
        """删除会话（级联删除任务）"""
        ...
    
    async def append_message(
        self,
        session_id: str,
        message: dict
    ) -> None:
        """追加消息到会话"""
        ...
```

### 4.2 Task Store

```python
class TaskStore:
    """任务存储接口"""
    
    async def create_task(
        self,
        session_id: str,
        paper_path: str
    ) -> Task:
        """创建任务"""
        ...
    
    async def get_task(self, task_id: str) -> Task | None:
        """获取任务"""
        ...
    
    async def list_tasks(
        self,
        session_id: str | None = None,
        status: TaskStatus | None = None
    ) -> list[Task]:
        """列出任务"""
        ...
    
    async def update_task(
        self,
        task_id: str,
        **kwargs
    ) -> Task:
        """更新任务"""
        ...
    
    async def delete_task(self, task_id: str) -> bool:
        """删除任务（保留文件）"""
        ...
```

---

## 5. Log Management

### 5.1 Log Structure

```
deepcode_lab/tasks/<task_id>/logs/
├── system.jsonl     # 系统日志（启动、阶段切换）
├── llm.jsonl        # LLM 调用日志（请求、响应）
└── mcp.jsonl        # MCP 工具调用日志
```

### 5.2 Log Entry Format

```python
# system.jsonl
{"timestamp": "2026-05-12T10:00:00Z", "level": "INFO", "event": "task_started", "task_id": "xxx"}
{"timestamp": "2026-05-12T10:00:01Z", "level": "INFO", "event": "phase_changed", "from": "PLANNING", "to": "IMPLEMENTATION"}

# llm.jsonl
{"timestamp": "2026-05-12T10:00:00Z", "model": "gpt-4o", "input_tokens": 1500, "output_tokens": 300, "cost": 0.05}
{"timestamp": "2026-05-12T10:00:05Z", "model": "gpt-4o", "input_tokens": 1800, "output_tokens": 450, "cost": 0.07}

# mcp.jsonl
{"timestamp": "2026-05-12T10:00:02Z", "tool": "read_file", "args": {"path": "/a/b/c.py"}, "duration_ms": 15}
{"timestamp": "2026-05-12T10:00:03Z", "tool": "write_file", "args": {"path": "/x/y.py"}, "result": "success"}
```

### 5.3 Context Variable Injection

```python
from contextvars import ContextVar

# task_id 自动注入到日志
task_id_var: ContextVar[str | None] = ContextVar("task_id", default=None)

# 在工作流中使用
def log_llm_call(model: str, usage: dict):
    task_id = task_id_var.get()
    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "task_id": task_id,
        "model": model,
        **usage
    }
    logger.info(log_entry)
```

---

## 6. Session Operations

### 6.1 Create Session

```bash
# CLI
python cli/main_cli.py session new

# API
POST /api/v1/sessions
{
    "title": "My Research Paper",
    "model": "gpt-4o"
}
```

### 6.2 List Sessions

```bash
# CLI
python cli/main_cli.py session list

# API
GET /api/v1/sessions?status=active&limit=10
```

### 6.3 Resume Session

```bash
# CLI
python cli/main_cli.py session resume <session_id>

# API
POST /api/v1/sessions/<session_id>/resume
```

### 6.4 Delete Session (Cascade)

```bash
# CLI
python cli/main_cli.py session delete <session_id>

# API
DELETE /api/v1/sessions/<session_id>

# Response
{
    "success": true,
    "deleted": {
        "session": "session_xxx",
        "tasks": ["task_001", "task_002"],
        "files_space_freed": "15MB"
    }
}
```

---

## 7. WebSocket Integration

### 7.1 Session Log Streaming

```python
# WebSocket /ws/sessions/{session_id}/logs
# 合并所有任务的日志

async def stream_session_logs(session_id: str):
    tasks = await task_store.list_tasks(session_id=session_id)
    
    async def follow_logs(task_id: str, channel: str):
        log_file = get_task_log_path(task_id, channel)
        async for line in tail_file(log_file):
            yield f"data: {line}\n\n"
    
    # 合并多个任务的日志流
    async def merged_stream():
        for task in tasks:
            for channel in ["system", "llm", "mcp"]:
                async for entry in follow_logs(task.id, channel):
                    yield entry
    
    return Response(
        content=merged_stream(),
        media_type="text/event-stream"
    )
```

### 7.2 Task Log Streaming

```python
# WebSocket /ws/tasks/{task_id}/logs?channel=llm

# Query params:
# - channel: system | llm | mcp
# - follow: true | false (是否持续跟踪)
```

---

## 8. Cleanup & Retention

### 8.1 Cascade Delete

```python
async def delete_session(self, session_id: str) -> DeleteResult:
    """删除会话及所有关联数据"""
    
    # 1. 获取所有任务
    tasks = await self.task_store.list_tasks(session_id=session_id)
    
    # 2. 删除任务工作区（保留 uploads/）
    for task in tasks:
        task_dir = get_task_dir(task.id)
        workspace_dir = task_dir / "workspace"
        if workspace_dir.exists():
            shutil.rmtree(workspace_dir)
    
    # 3. 删除 session.jsonl
    session_file = self.sessions_dir / session_id / "session.jsonl"
    session_file.unlink(missing_ok=True)
    
    # 4. 删除会话目录
    session_dir = self.sessions_dir / session_id
    session_dir.rmdir()  # 目录应为空
    
    return DeleteResult(
        session_id=session_id,
        tasks_deleted=len(tasks),
        space_freed=estimated_size
    )
```

### 8.2 Blocking Conditions

```python
# 禁止删除有 pending/running/waiting 状态任务的会话
BLOCKED_STATUSES = {
    TaskStatus.PENDING,
    TaskStatus.RUNNING,
    TaskStatus.WAITING_FOR_INPUT
}

async def delete_session(self, session_id: str) -> None:
    tasks = await self.task_store.list_tasks(session_id=session_id)
    
    blocked = [t for t in tasks if t.status in BLOCKED_STATUSES]
    if blocked:
        raise SessionDeleteBlocked(
            f"Cannot delete session with {len(blocked)} active tasks",
            active_tasks=blocked
        )
```

---

## 9. Configuration

### 9.1 Config Schema

```json
{
  "sessions": {
    "dir": "~/.deepcode/sessions",
    "max_idle_days": 30,
    "auto_cleanup": true
  },
  "logs": {
    "global_file": "logs/server-YYYYMMDD.jsonl",
    "task_file": "deepcode_lab/tasks/{task_id}/logs/{channel}.jsonl",
    "retention_days": 90
  },
  "workspace": {
    "preserve_on_completion": true,
    "uploads_shared": true
  }
}
```

### 9.2 Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DEEPCODE_SESSIONS_DIR` | `~/.deepcode/sessions` | 会话根目录 |
| `DEEPCODE_LABS_DIR` | `~/.deepcode/deepcode_lab` | 工作区根目录 |

---

## 10. API Reference

### 10.1 Sessions API

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/sessions` | 创建会话 |
| `GET` | `/api/v1/sessions` | 列出会话 |
| `GET` | `/api/v1/sessions/{id}` | 获取会话 |
| `PUT` | `/api/v1/sessions/{id}` | 更新会话 |
| `DELETE` | `/api/v1/sessions/{id}` | 删除会话 |
| `POST` | `/api/v1/sessions/{id}/resume` | 恢复会话 |

### 10.2 Tasks API

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/tasks` | 创建任务 |
| `GET` | `/api/v1/tasks/{id}` | 获取任务 |
| `GET` | `/api/v1/tasks/{id}/status` | 获取任务状态 |
| `PUT` | `/api/v1/tasks/{id}/abort` | 中止任务 |

---

## 11. Implementation Notes

### 11.1 Key Files

| File | Purpose |
|------|---------|
| `core/sessions/store.py` | 会话存储实现 (18KB) |
| `core/sessions/models.py` | 数据模型定义 |
| `core/sessions/__init__.py` | 接口导出 |

### 11.2 Known Constraints

- Session 删除仅清理 workspace/，uploads/ 保留
- 任务处于 pending/running/waiting 时禁止删除会话
- 日志使用 contextvar 自动注入 task_id，业务代码无需手动传递
- 旧版 `services/session_service.py` 已废弃（2026-04-28）

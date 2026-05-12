# Tool System Architecture

> DeepCode MCP (Model Context Protocol) 工具系统架构设计

## 1. Overview

DeepCode 采用 MCP (Model Context Protocol) 实现工具调用标准化，将代码实现、文档分割等功能封装为独立的 MCP Server，通过统一接口调用。

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Tool System Architecture                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                    MCP Client (Agent Runtime)                    │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │ │
│  │  │ tool_call() │  │  results()  │  │   list()   │            │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘            │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                              ↓                                       │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                    MCP Server Layer                              │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │ │
│  │  │ Code Impl  │  │   Doc Seg   │  │  Git Ops   │            │ │
│  │  │  Server    │  │   Server   │  │  Server    │            │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘            │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                              ↓                                       │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                    Native Tool Layer                            │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │ │
│  │  │ read_file  │  │ write_file  │  │   terminal │            │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘            │ │
│  └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. MCP Architecture

### 2.1 MCP Server Implementations

| Server | File | Purpose | Key Tools |
|--------|------|---------|-----------|
| Code Implementation Server | `tools/code_implementation_server.py` | 代码生成与迭代 | `generate_file`, `implement_file`, `validate_syntax` |
| Document Segmentation Server | `tools/document_segmentation_server.py` | 论文解析与分割 | `segment_paper`, `extract_sections` |
| Git Operations Server | `tools/git_server.py` | Git 仓库操作 | `clone_repo`, `checkout_branch` |
| File Operations Server | `tools/file_server.py` | 文件读写操作 | `read_file`, `write_file`, `list_dir` |

### 2.2 MCP Protocol

```python
# MCP Message Types
class MCPMessageType(Enum):
    TOOL_CALL = "tool_call"
    TOOL_RESULT = "tool_result"
    LIST_TOOLS = "list_tools"
    TOOL_SCHEMA = "tool_schema"

# Tool Call Request
@dataclass
class ToolCallRequest:
    id: str                    # Call ID
    name: str                  # Tool name
    arguments: dict[str, Any]  # Arguments

# Tool Result Response
@dataclass
class ToolResult:
    id: str                    # Corresponds to call ID
    success: bool
    result: Any | None
    error: str | None
    duration_ms: float
```

---

## 3. Code Implementation Server

### 3.1 Server Overview

```python
# tools/code_implementation_server.py
class CodeImplementationServer:
    """代码实现 MCP Server"""
    
    TOOLS = [
        "code_generation",
        "file_implementation",
        "syntax_validation",
        "dependency_resolution",
        "test_generation"
    ]
```

### 3.2 Core Tools

#### code_generation

```python
async def code_generation(
    file_path: str,
    requirements: str,
    context: dict | None = None,
    model: str = "gpt-4o"
) -> dict:
    """
    Generate code for a single file.
    
    Args:
        file_path: Target file path
        requirements: File description and requirements
        context: Additional context (dependencies, imports)
        model: LLM model to use
    
    Returns:
        {
            "success": bool,
            "code": str,
            "language": str,
            "validation": dict
        }
    """
```

#### syntax_validation

```python
async def syntax_validation(
    code: str,
    language: str
) -> dict:
    """
    Validate code syntax.
    
    Args:
        code: Code content
        language: Programming language
    
    Returns:
        {
            "valid": bool,
            "errors": list[SyntaxError],
            "warnings": list[SyntaxWarning]
        }
    """
```

### 3.3 File Implementation Flow

```python
async def implement_file(
    spec: FileSpec,
    workspace: Path,
    models: dict[str, str]
) -> ImplementationResult:
    """实现单个文件的完整流程"""
    
    # 1. 读取上下文（依赖文件）
    context = await _load_context(spec.dependencies, workspace)
    
    # 2. 生成代码
    code = await code_generation(
        file_path=spec.path,
        requirements=spec.description,
        context=context,
        model=models.get("implementation", "gpt-4o")
    )
    
    # 3. 验证语法
    validation = await syntax_validation(code, spec.language)
    if not validation["valid"]:
        # 重试或报错
        pass
    
    # 4. 写入文件
    await write_file(workspace / spec.path, code)
    
    return ImplementationResult(success=True, path=spec.path)
```

---

## 4. Document Segmentation Server

### 4.1 Server Overview

```python
# tools/document_segmentation_server.py
class DocumentSegmentationServer:
    """文档分割 MCP Server"""
    
    CHUNK_SIZE = 50000  # 字符数
    
    TOOLS = [
        "segment_paper",
        "extract_abstract",
        "extract_figures",
        "extract_tables",
        "extract_equations"
    ]
```

### 4.2 Core Tools

#### segment_paper

```python
async def segment_paper(
    paper_path: str,
    chunk_size: int = 50000,
    overlap: int = 500
) -> dict:
    """
    Segment a paper into chunks.
    
    Args:
        paper_path: Path to paper file
        chunk_size: Maximum chunk size in characters
        overlap: Overlap between chunks
    
    Returns:
        {
            "segments": list[Segment],
            "total_chars": int,
            "num_segments": int
        }
    """
```

#### extract_abstract

```python
async def extract_abstract(paper_content: str) -> dict:
    """
    Extract abstract from paper content.
    
    Returns:
        {
            "abstract": str,
            "section_ref": str
        }
    """
```

### 4.3 Segmentation Strategy

```python
class SemanticSegmenter:
    """语义分割器"""
    
    def segment(self, content: str, chunk_size: int) -> list[str]:
        # 1. 尝试按章节分割
        chapters = self._split_by_headings(content)
        if len(chapters) > 1:
            return self._balance_chapters(chapters, chunk_size)
        
        # 2. 按段落分割
        paragraphs = self._split_by_paragraphs(content)
        return self._group_paragraphs(paragraphs, chunk_size)
    
    def _balance_chapters(self, chapters: list[str], target_size: int) -> list[str]:
        """平衡章节大小"""
        current_chunk = []
        current_size = 0
        chunks = []
        
        for chapter in chapters:
            if current_size + len(chapter) > target_size and current_chunk:
                chunks.append("\n".join(current_chunk))
                current_chunk = []
                current_size = 0
            current_chunk.append(chapter)
            current_size += len(chapter)
        
        if current_chunk:
            chunks.append("\n".join(current_chunk))
        
        return chunks
```

---

## 5. Tool Registry

### 5.1 Registry Architecture

```python
class ToolRegistry:
    """工具注册表"""
    
    def __init__(self):
        self._tools: dict[str, ToolSpec] = {}
        self._servers: dict[str, MCPServer] = {}
    
    def register_tool(self, name: str, spec: ToolSpec, server: MCPServer):
        """注册工具"""
        self._tools[name] = spec
        self._servers[name] = server
    
    def get_tool(self, name: str) -> ToolSpec | None:
        """获取工具规范"""
        return self._tools.get(name)
    
    def list_tools(self) -> list[str]:
        """列出所有工具"""
        return list(self._tools.keys())
    
    async def call_tool(self, name: str, **kwargs) -> ToolResult:
        """调用工具"""
        tool = self._tools.get(name)
        if not tool:
            raise ToolNotFoundError(name)
        
        server = self._servers[name]
        return await server.call_tool(name, **kwargs)
```

### 5.2 Tool Spec

```python
@dataclass
class ToolSpec:
    """工具规范"""
    name: str
    description: str
    input_schema: dict  # JSON Schema
    output_schema: dict  # JSON Schema
    server: str          # MCP Server name
    retryable: bool = True
    timeout_ms: int = 30000
```

---

## 6. Tool Execution

### 6.1 Execution Context

```python
@dataclass
class ToolContext:
    """工具执行上下文"""
    task_id: str
    session_id: str
    workspace: Path
    models: dict[str, str]
    config: DeepCodeConfig
    progress: ProgressTracker
```

### 6.2 Retry Strategy

```python
class ToolRetryStrategy:
    """工具重试策略"""
    
    @staticmethod
    def should_retry(tool_name: str, error: Exception) -> bool:
        # 网络错误重试
        if isinstance(error, (TimeoutError, ConnectionError)):
            return True
        
        # 速率限制重试
        if isinstance(error, RateLimitError):
            return True
        
        # 语法错误不重试
        if isinstance(error, SyntaxError):
            return False
        
        return False
    
    @staticmethod
    def get_delay(attempt: int, error: Exception) -> float:
        if isinstance(error, RateLimitError):
            # 指数退避
            return min(2 ** attempt, 60)
        return 0.5 * attempt
```

### 6.3 Timeout Handling

```python
async def call_with_timeout(
    coro: Coroutine,
    timeout_ms: int
) -> Any:
    """带超时的调用"""
    try:
        return await asyncio.wait_for(coro, timeout=timeout_ms / 1000)
    except asyncio.TimeoutError:
        raise ToolTimeoutError(f"Tool call exceeded {timeout_ms}ms")
```

---

## 7. Result Persistence

### 7.1 MCP Log

```python
async def log_mcp_call(
    task_id: str,
    tool: str,
    args: dict,
    result: Any,
    duration_ms: float,
    error: str | None = None
) -> None:
    """记录 MCP 调用到日志"""
    log_dir = Path(f"~/.deepcode/sessions/{session_id}/tasks/{task_id}/logs")
    log_dir.mkdir(parents=True, exist_ok=True)
    
    log_file = log_dir / "mcp.jsonl"
    entry = {
        "timestamp": datetime.now().isoformat(),
        "task_id": task_id,
        "tool": tool,
        "args": args,
        "duration_ms": duration_ms,
        "success": error is None,
        "error": error
    }
    
    async with aiofiles.open(log_file, "a") as f:
        await f.write(json.dumps(entry) + "\n")
```

### 7.2 Log Format

```json
{"timestamp": "2026-05-12T10:00:00Z", "tool": "code_generation", "args": {"file_path": "a.py"}, "duration_ms": 1500, "success": true}
{"timestamp": "2026-05-12T10:00:02Z", "tool": "syntax_validation", "args": {"code": "..."}, "duration_ms": 50, "success": true}
{"timestamp": "2026-05-12T10:00:03Z", "tool": "segment_paper", "args": {"paper_path": "paper.pdf"}, "duration_ms": 3000, "success": false, "error": "File not found"}
```

---

## 8. Configuration

### 8.1 Tool Config Schema

```json
{
  "tools": {
    "code_implementation": {
      "enabled": true,
      "timeout_ms": 60000,
      "retry": {
        "max_attempts": 3,
        "backoff": "exponential"
      }
    },
    "document_segmentation": {
      "enabled": true,
      "chunk_size": 50000,
      "overlap": 500
    },
    "git_operations": {
      "enabled": true,
      "timeout_ms": 30000
    }
  },
  "mcp": {
    "servers": {
      "code_implementation": {
        "command": "python",
        "args": ["-m", "tools.code_implementation_server"]
      },
      "document_segmentation": {
        "command": "python",
        "args": ["-m", "tools.document_segmentation_server"]
      }
    }
  }
}
```

---

## 9. API Reference

### 9.1 Tool Listing

```python
# GET /api/v1/tools
{
    "tools": [
        {
            "name": "code_generation",
            "description": "Generate code for a file",
            "server": "code_implementation",
            "input_schema": {...},
            "output_schema": {...}
        }
    ]
}
```

### 9.2 Tool Call

```python
# POST /api/v1/tools/call
{
    "tool": "code_generation",
    "arguments": {
        "file_path": "src/models/transformer.py",
        "requirements": "...",
        "model": "gpt-4o"
    }
}

# Response
{
    "success": true,
    "result": {
        "code": "...",
        "language": "python"
    },
    "duration_ms": 1500
}
```

---

## 10. Implementation Notes

### 10.1 Key Files

| File | Purpose | Size |
|------|---------|------|
| `tools/code_implementation_server.py` | 代码实现 Server | 73KB |
| `tools/document_segmentation_server.py` | 文档分割 Server | 73KB |
| `tools/git_server.py` | Git 操作 Server | 30KB |
| `tools/file_server.py` | 文件操作 Server | 25KB |
| `core/agent_runtime/tools/registry.py` | 工具注册表 | 5KB |

### 10.2 Known Constraints

- MCP Server 必须在工作流启动前初始化
- 工具调用超时后自动重试（可配置）
- 结果持久化到 `mcp.jsonl`，不保存完整 code 内容
- 文件操作工具仅限 workspace 目录内，防止路径遍历

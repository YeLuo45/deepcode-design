# Provider Architecture

> DeepCode LLM Provider 抽象层架构设计

## 1. Overview

DeepCode 通过统一的 Provider 抽象层，支持多种 LLM 后端（OpenAI、Anthropic、OpenRouter 等），实现模型切换的热插拔。

```
┌─────────────────────────────────────────────────────────────┐
│                    Provider Architecture                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  LLMProvider (ABC)                    │   │
│  │  - generate()                                         │   │
│  │  - stream()                                           │   │
│  │  - get_usage()                                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                          ↑                                   │
│         ┌───────────────┼───────────────┐                   │
│         ↓               ↓               ↓                   │
│  ┌──────────┐    ┌───────────┐   ┌───────────┐           │
│  │  OpenAI  │    │ Anthropic │   │ OpenRouter │           │
│  │ Provider │    │  Provider │   │  Provider  │           │
│  └──────────┘    └───────────┘   └───────────┘           │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Provider Registry                        │   │
│  │  - register(provider_name, provider_class)           │   │
│  │  - get_provider(provider_name) -> LLMProvider       │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Provider Interface

### 2.1 LLMProvider (Abstract Base Class)

```python
class LLMProvider(ABC):
    """LLM Provider 抽象接口"""
    
    @abstractmethod
    async def generate(
        self,
        messages: list[dict],
        model: str,
        temperature: float = 0.7,
        max_tokens: int | None = None,
        **kwargs
    ) -> LLMResponse:
        """同步生成"""
        pass
    
    @abstractmethod
    async def stream(
        self,
        messages: list[dict],
        model: str,
        **kwargs
    ) -> AsyncIterator[LLMResponse]:
        """流式生成"""
        pass
    
    def get_usage(self) -> dict[str, int]:
        """获取用量统计"""
        pass
```

### 2.2 LLMResponse

```python
@dataclass
class LLMResponse:
    """LLM 响应封装"""
    content: str | None
    tool_calls: list[ToolCallRequest] = field(default_factory=list)
    finish_reason: str = "stop"
    usage: dict[str, int] = field(default_factory=dict)
    retry_after: float | None = None
    reasoning_content: str | None = None
    thinking_blocks: list[dict] | None = None
    
    @property
    def has_tool_calls(self) -> bool:
        return len(self.tool_calls) > 0
    
    @property
    def should_execute_tools(self) -> bool:
        if not self.has_tool_calls:
            return False
        return self.finish_reason in ("tool_calls", "stop")
```

### 2.3 ToolCallRequest

```python
@dataclass
class ToolCallRequest:
    """工具调用请求"""
    id: str
    name: str
    arguments: dict[str, Any]
    extra_content: dict[str, Any] | None = None
    provider_specific_fields: dict[str, Any] | None = None
    
    def to_openai_tool_call(self) -> dict[str, Any]:
        """序列化为 OpenAI 格式"""
        ...
```

---

## 3. Provider Implementations

### 3.1 OpenAI Provider

```python
class OpenAIProvider(LLMProvider):
    """OpenAI API 提供商"""
    
    DEFAULT_MODELS = {
        "gpt-4o": {"context_window": 128000, "max_output": 4096},
        "gpt-4o-mini": {"context_window": 128000, "max_output": 16384},
        "gpt-4-turbo": {"context_window": 128000, "max_output": 4096},
    }
    
    def __init__(self, api_key: str, base_url: str | None = None):
        self.client = OpenAI(api_key=api_key, base_url=base_url)
```

### 3.2 Anthropic Provider

```python
class AnthropicProvider(LLMProvider):
    """Anthropic Claude 提供商"""
    
    DEFAULT_MODELS = {
        "claude-sonnet-4-20250514": {"context_window": 200000, "max_output": 8192},
        "claude-opus-4-20250514": {"context_window": 200000, "max_output": 8192},
    }
    
    def __init__(self, api_key: str):
        self.client = Anthropic(api_key=api_key)
```

### 3.3 OpenRouter Provider

```python
class OpenRouterProvider(LLMProvider):
    """OpenRouter 聚合提供商"""
    
    # 支持 100+ 模型
    SUPPORTED_MODELS = [
        "anthropic/claude-sonnet-4",
        "z-ai/glm-5.1",
        "openai/gpt-4o",
        # ... 更多模型
    ]
    
    async def fetch_model_list(self) -> list[dict]:
        """从 OpenRouter API 获取可用模型"""
        response = await self.client.get("/v1/models")
        return response.json()["data"]
```

---

## 4. Provider Registry

### 4.1 Registry Pattern

```python
class ProviderRegistry:
    """LLM Provider 注册表"""
    
    _providers: dict[str, type[LLMProvider]] = {}
    
    @classmethod
    def register(cls, name: str, provider_class: type[LLMProvider]):
        """注册 Provider"""
        cls._providers[name] = provider_class
    
    @classmethod
    def get_provider(cls, name: str) -> LLMProvider:
        """获取 Provider 实例"""
        if name not in cls._providers:
            raise ValueError(f"Unknown provider: {name}")
        return cls._providers[name]()
    
    @classmethod
    def list_providers(cls) -> list[str]:
        """列出所有可用 Provider"""
        return list(cls._providers.keys())
```

### 4.2 Supported Providers

| Provider | Config Key | Models | Status |
|----------|------------|--------|--------|
| OpenAI | `openai.api_key` | gpt-4o, gpt-4o-mini, gpt-4-turbo | ✅ |
| Anthropic | `anthropic.api_key` | claude-sonnet-4, claude-opus-4 | ✅ |
| OpenRouter | `openrouter.api_key` | 100+ models | ✅ |
| OpenAI Compatible | `openai.base_url` | Any OpenAI-compatible API | ✅ |

---

## 5. Model Selection

### 5.1 Model Configuration

```python
@dataclass
class ModelSelection:
    """模型选择配置"""
    default: str                    # 默认模型
    planning: str                   # 规划阶段模型
    implementation: str             # 实现阶段模型
    provider: str                   # Provider 名称
```

### 5.2 Dynamic Model Limits

```python
# tools/model_limits.py
class ModelLimits:
    """动态检测和适配模型 token 限制"""
    
    @classmethod
    async def detect_limits(cls, model: str, provider: str) -> ModelLimits:
        """检测模型限制"""
        # 从配置或 API 获取
        limits = cls._load_from_config(model)
        if not limits:
            limits = await cls._fetch_from_provider(model, provider)
        return limits
    
    @classmethod
    def adapt_max_tokens(cls, model: str, max_tokens: int) -> int:
        """适配 max_tokens"""
        limits = cls.get_limits(model)
        # 确保不超过模型最大输出
        return min(max_tokens, limits.max_output)
```

### 5.3 Token Budget Calculation

```python
class TokenBudget:
    """Token 预算计算"""
    
    def __init__(self, model: str, context_window: int):
        self.model = model
        self.context_window = context_window
        self.reserved = 1024  # 安全缓冲
    
    @property
    def available(self) -> int:
        """可用 token 数量"""
        return self.context_window - self.reserved
    
    def calculate_max_input(self, output_tokens: int) -> int:
        """计算最大输入 token"""
        return self.available - output_tokens
```

---

## 6. Runtime Integration

### 6.1 Attaching Provider to Agent

```python
def attach_workflow_llm(
    agent: Agent,
    provider: LLMProvider,
    model: str
) -> Agent:
    """为 Agent 绑定 Provider"""
    agent._llm_provider = provider
    agent._model = model
    return agent

def get_workflow_provider() -> LLMProvider:
    """获取当前工作流 Provider"""
    return _workflow_provider.get()
```

### 6.2 Retry Logic

```python
class RetryConfig:
    """重试配置"""
    
    MODES = {
        "standard": {
            "max_retries": 3,
            "backoff_factor": 1.5,
            "retry_on": ["rate_limit", "timeout", "server_error"]
        },
        "aggressive": {
            "max_retries": 5,
            "backoff_factor": 1.0,
            "retry_on": ["rate_limit", "timeout", "server_error", "model_error"]
        }
    }
```

---

## 7. Cost Tracking

### 7.1 Usage Tracking

```python
@dataclass
class UsageRecord:
    """用量记录"""
    model: str
    input_tokens: int
    output_tokens: int
    cost: float
    timestamp: datetime

class CostTracker:
    """成本追踪"""
    
    PRICING = {
        "gpt-4o": {"input": 2.5, "output": 10.0},  # $ / 1M tokens
        "claude-sonnet-4": {"input": 3.0, "output": 15.0},
    }
    
    def record(self, model: str, usage: dict[str, int]):
        """记录用量"""
        cost = self.calculate_cost(model, usage)
        self._records.append(UsageRecord(model, **usage, cost=cost))
    
    def calculate_cost(self, model: str, usage: dict[str, int]) -> float:
        """计算成本"""
        prices = self.PRICING.get(model, {"input": 0, "output": 0})
        return (
            usage["input_tokens"] * prices["input"] / 1_000_000 +
            usage["output_tokens"] * prices["output"] / 1_000_000
        )
```

---

## 8. Configuration

### 8.1 Config Schema

```json
{
  "openai": {
    "api_key": "sk-...",
    "base_url": "https://api.openai.com/v1"
  },
  "anthropic": {
    "api_key": "sk-ant-..."
  },
  "openrouter": {
    "api_key": "sk-or-..."
  },
  "agents": {
    "defaults": {
      "model": "gpt-4o",
      "provider": "openai",
      "temperature": 0.7
    },
    "planning": {
      "model": "gpt-4o"
    },
    "implementation": {
      "model": "claude-sonnet-4"
    }
  }
}
```

---

## 9. Implementation Notes

### 9.1 Key Files

| File | Purpose |
|------|---------|
| `core/providers/base.py` | Provider 抽象基类 (27KB) |
| `core/providers/openai_compat.py` | OpenAI 兼容 Provider (50KB) |
| `core/providers/anthropic.py` | Anthropic Provider (24KB) |
| `core/providers/registry.py` | Provider 注册表 |
| `core/llm_runtime.py` | LLM 运行时集成 |
| `tools/model_limits.py` | 动态模型限制检测 |

### 9.2 Known Constraints

- OpenRouter 模型列表需要网络请求，建议缓存
- 部分 Provider 不支持 streaming，需要降级
- provider_retry_mode 需要在 AgentRunSpec 中指定

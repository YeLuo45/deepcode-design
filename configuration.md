# Configuration Management

> DeepCode 配置管理架构设计

## 1. Overview

DeepCode 采用 YAML 配置文件管理所有运行时参数，支持环境变量覆盖、多层级配置合并。

```
~/.deepcode/
├── config.yaml              # 主配置文件
├── deepcode_config.json    # Provider 配置 (API Keys)
├── .env                     # 环境变量
└── sessions/               # 会话数据
```

---

## 2. Config Files

### 2.1 Main Config (config.yaml)

```yaml
# DeepCode 主配置文件
version: "1.0"

# LLM Provider 配置
providers:
  openai:
    api_key: ${OPENAI_API_KEY}
    base_url: https://api.openai.com/v1
    timeout: 120
  anthropic:
    api_key: ${ANTHROPIC_API_KEY}
    timeout: 120
  openrouter:
    api_key: ${OPENROUTER_API_KEY}
    base_url: https://openrouter.ai/api/v1
    timeout: 120

# Agent 模型选择
agents:
  defaults:
    model: gpt-4o
    provider: openai
    temperature: 0.7
  planning:
    model: gpt-4o
    provider: openai
  implementation:
    model: claude-sonnet-4-20250514
    provider: anthropic

# 工作流配置
workflow:
  max_plan_retries: 3
  enable_plan_review: true
  loop_detection:
    max_consecutive_errors: 10
    max_iterations: 50
    stall_threshold_s: 300

# 会话配置
sessions:
  dir: ~/.deepcode/sessions
  max_idle_days: 30
  auto_cleanup: true

# 日志配置
logs:
  level: INFO
  format: json
  retention_days: 90
```

### 2.2 Environment Variables

```bash
# .env file
OPENAI_API_KEY=sk-xxxxx
ANTHROPIC_API_KEY=sk-ant-xxxxx
OPENROUTER_API_KEY=sk-or-xxxxx

# 可选覆盖
DEEPCODE_CONFIG_PATH=~/.deepcode/config.yaml
DEEPCODE_SESSIONS_DIR=~/.deepcode/sessions
```

---

## 3. Configuration Models

### 3.1 DeepCodeConfig

```python
@dataclass
class DeepCodeConfig:
    """主配置数据类"""
    version: str
    providers: dict[str, ProviderConfig]
    agents: AgentConfig
    workflow: WorkflowConfig
    sessions: SessionConfig
    logs: LogConfig
    
    @classmethod
    def from_yaml(cls, path: Path) -> "DeepCodeConfig":
        """从 YAML 文件加载"""
        with open(path) as f:
            data = yaml.safe_load(f)
        return cls(**data)
    
    @classmethod
    def from_env(cls) -> "DeepCodeConfig":
        """从环境变量加载"""
        # 从 env 文件加载
        load_dotenv()
        
        # 构建配置
        ...
```

### 3.2 ProviderConfig

```python
@dataclass
class ProviderConfig:
    """Provider 配置"""
    api_key: str
    base_url: str | None = None
    timeout: int = 120
    retry_mode: str = "standard"
    max_retries: int = 3
    
    @property
    def is_available(self) -> bool:
        """检查是否配置了 API Key"""
        return bool(self.api_key)
```

### 3.3 AgentConfig

```python
@dataclass
class AgentConfig:
    """Agent 配置"""
    defaults: ModelSelection
    planning: ModelSelection
    implementation: ModelSelection
    reasoning_effort: str | None = None
    
@dataclass
class ModelSelection:
    """模型选择"""
    model: str
    provider: str
    temperature: float = 0.7
    max_tokens: int | None = None
```

---

## 4. Config Loading

### 4.1 Loading Priority

```
Environment Variables (highest)
    ↓
CLI Arguments
    ↓
~/.deepcode/config.yaml
    ↓
Default Values (lowest)
```

### 4.2 Config Loader

```python
class ConfigLoader:
    """配置加载器"""
    
    DEFAULT_CONFIG_PATH = Path("~/.deepcode/config.yaml")
    
    def load(self) -> DeepCodeConfig:
        """加载配置"""
        # 1. 加载默认配置
        config = self._load_default()
        
        # 2. 合并 YAML 配置
        yaml_path = self._get_config_path()
        if yaml_path.exists():
            yaml_config = self._load_yaml(yaml_path)
            config = self._merge(config, yaml_config)
        
        # 3. 合并环境变量
        env_config = self._load_env()
        config = self._merge(config, env_config)
        
        return config
    
    def _load_env(self) -> dict:
        """从环境变量加载"""
        return {
            "providers": {
                "openai": {"api_key": os.getenv("OPENAI_API_KEY")},
                "anthropic": {"api_key": os.getenv("ANTHROPIC_API_KEY")},
            }
        }
```

---

## 5. Provider Key Management

### 5.1 API Key Storage

```python
# 使用 keyring 安全存储
import keyring

class APIKeyManager:
    """API Key 管理器"""
    
    SERVICE_NAME = "deepcode"
    
    @staticmethod
    def save_key(provider: str, key: str) -> None:
        """保存 API Key"""
        keyring.set_password(SERVICE_NAME, provider, key)
    
    @staticmethod
    def get_key(provider: str) -> str | None:
        """获取 API Key"""
        return keyring.get_password(SERVICE_NAME, provider)
    
    @staticmethod
    def delete_key(provider: str) -> None:
        """删除 API Key"""
        keyring.delete_password(SERVICE_NAME, provider)
```

### 5.2 Interactive Setup

```bash
# 首次设置引导
$ deepcode config setup

Welcome to DeepCode configuration!
Enter your API keys (press Enter to skip):

OpenAI API Key: sk-xxxxx
Anthropic API Key: sk-ant-xxxxx
OpenRouter API Key (optional): sk-or-xxxxx

Configuration saved to ~/.deepcode/config.yaml
```

---

## 6. Model Limits

### 6.1 Dynamic Limit Detection

```python
# tools/model_limits.py
class ModelLimits:
    """动态模型限制检测"""
    
    # 静态配置 (默认限制)
    STATIC_LIMITS = {
        "gpt-4o": {"context_window": 128000, "max_output": 4096},
        "gpt-4o-mini": {"context_window": 128000, "max_output": 16384},
        "claude-sonnet-4-20250514": {"context_window": 200000, "max_output": 8192},
    }
    
    @classmethod
    async def get_limits(cls, model: str, provider: str) -> dict:
        """获取模型限制"""
        # 1. 先查缓存
        cached = cls._get_from_cache(model)
        if cached:
            return cached
        
        # 2. 尝试从 API 获取
        if provider == "openai":
            limits = await cls._fetch_openai_limits(model)
        elif provider == "anthropic":
            limits = cls.STATIC_LIMITS.get(model, {})
        else:
            limits = cls.STATIC_LIMITS.get(model, {})
        
        # 3. 缓存并返回
        cls._cache_limits(model, limits)
        return limits
```

### 6.2 Token Budget Calculation

```python
class TokenBudget:
    """Token 预算计算"""
    
    def __init__(self, model: str):
        limits = ModelLimits.get_limits(model)
        self.context_window = limits["context_window"]
        self.max_output = limits["max_output"]
        self.reserved = 1024  # 安全缓冲
    
    @property
    def available_input(self) -> int:
        """可用输入 token"""
        return self.context_window - self.max_output - self.reserved
    
    def calculate_safe_max_tokens(self, estimated_output: int) -> int:
        """计算安全的最大输出 token"""
        return min(estimated_output, self.max_output)
```

---

## 7. Workspace Configuration

### 7.1 Workspace Layout

```yaml
# .deepcode/workspace.yaml
workspace:
  layout: |
    {task_id}/
    ├── logs/
    │   ├── system.jsonl
    │   ├── llm.jsonl
    │   └── mcp.jsonl
    ├── workspace/
    │   └── (generated files)
    └── metadata.json
  
  # 保留策略
  retention:
    on_success: keep  # keep | delete
    on_failure: keep
    max_age_days: 30
```

### 7.2 Per-Task Config

```python
@dataclass
class TaskConfig:
    """任务级配置"""
    task_id: str
    session_id: str
    paper_path: str
    output_dir: Path
    models: dict[str, str]
    workflow_options: dict
    environment: dict[str, str]
```

---

## 8. CLI Config Commands

### 8.1 Config Management

```bash
# 查看当前配置
deepcode config show

# 编辑配置
deepcode config edit

# 设置 API Key
deepcode config set-key openai sk-xxxxx

# 验证配置
deepcode config validate

# 导出配置 (不含密钥)
deepcode config export --mask-secrets
```

### 8.2 Config Template

```bash
# 生成默认配置
deepcode config init --template minimal
deepcode config init --template full

# 输出示例
$ deepcode config init --template minimal

# ~/.deepcode/config.yaml
version: "1.0"
providers:
  openai:
    api_key: ${OPENAI_API_KEY}
agents:
  defaults:
    model: gpt-4o
    provider: openai
```

---

## 9. Validation

### 9.1 Config Validator

```python
class ConfigValidator:
    """配置验证器"""
    
    def validate(self, config: DeepCodeConfig) -> list[ValidationError]:
        errors = []
        
        # 1. 验证 API Keys
        for name, provider in config.providers.items():
            if not provider.api_key:
                errors.append(ValidationError(
                    field=f"providers.{name}.api_key",
                    message=f"Missing API key for {name}"
                ))
        
        # 2. 验证模型
        valid_models = self._get_supported_models()
        for agent_name, selection in config.agents:
            if selection.model not in valid_models:
                errors.append(ValidationError(
                    field=f"agents.{agent_name}.model",
                    message=f"Unsupported model: {selection.model}"
                ))
        
        # 3. 验证路径
        if not config.sessions.dir:
            errors.append(ValidationError(
                field="sessions.dir",
                message="Session directory not configured"
            ))
        
        return errors
```

### 9.2 Health Check

```bash
$ deepcode config doctor

Running health checks...
✓ Config file found: ~/.deepcode/config.yaml
✓ Session directory: ~/.deepcode/sessions (writable)
✓ OpenAI API key configured
✗ Anthropic API key missing
✓ Logs directory: ~/.deepcode/logs (writable)

Summary: 1 warning, 0 errors
```

---

## 10. Implementation Notes

### 10.1 Key Files

| File | Purpose |
|------|---------|
| `core/config.py` | 配置加载和验证 |
| `core/config_models.py` | 配置数据模型 |
| `tools/model_limits.py` | 动态模型限制 |
| `services/key_manager.py` | API Key 管理 |

### 10.2 Known Constraints

- API Keys 优先从环境变量读取，其次是 config.yaml
- 配置变更需要重启服务生效
- 模型限制缓存 24 小时后刷新
- 不支持运行时配置热更新

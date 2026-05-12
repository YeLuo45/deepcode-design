# DeepCode Design

> DeepCode 多智能体代码生成系统的完整设计规范文档

## 📋 Document Index

### 🏗️ System Architecture

| Document | Description | Status |
|----------|-------------|--------|
| [SPEC.md](SPEC.md) | 项目设计规范主文档 | ✅ |
| [multi-agent-architecture.md](multi-agent-architecture.md) | 多智能体架构设计 | ✅ |
| [provider-architecture.md](provider-architecture.md) | LLM 提供商架构设计 | ✅ |
| [workflow-architecture.md](workflow-architecture.md) | 工作流架构设计 | ✅ |

### 📊 Data & State

| Document | Description | Status |
|----------|-------------|--------|
| [session-management.md](session-management.md) | 会话状态管理设计 | ✅ |
| [tool-system.md](tool-system.md) | 工具系统设计 | ✅ |
| [configuration.md](configuration.md) | 配置管理设计 | ✅ |

---

## 🤖 Agent System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DeepCode Multi-Agent System                         │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐  │
│  │  Research   │ │ Workspace   │ │   Code     │ │  Reference  │  │
│  │  Analysis   │ │Infrastructure│ │Architecture│ │ Intelligence│  │
│  │   Agent     │ │   Agent    │ │   Agent    │ │   Agent    │  │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                  │
│  │ Repository │ │ Codebase    │ │   Code     │                  │
│  │Acquisition │ │ Intelligence│ │Implementation│                  │
│  │   Agent    │ │   Agent    │ │   Agent    │                  │
│  └─────────────┘ └─────────────┘ └─────────────┘                  │
├─────────────────────────────────────────────────────────────────────┤
│                    Orchestration Engine                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📁 Project Structure

```
deepcode-design/
├── SPEC.md                              # 主规范文档
├── README.md                            # 本文档
├── index.html                          # Web 单页入口
├── multi-agent-architecture.md          # 多智能体架构
├── provider-architecture.md            # LLM 提供商架构
├── workflow-architecture.md             # 工作流架构
├── session-management.md               # 会话管理
├── tool-system.md                      # 工具系统
└── configuration.md                    # 配置管理
```

---

## 🔗 Related Links

- **DeepCode Repository**: https://github.com/HKUDS/DeepCode
- **Paper**: https://arxiv.org/abs/2512.07921
- **Web UI**: https://github.com/HKUDS/DeepCode/tree/main/new_ui
- **nanobot Integration**: https://github.com/HKUDS/DeepCode/tree/main/nanobot

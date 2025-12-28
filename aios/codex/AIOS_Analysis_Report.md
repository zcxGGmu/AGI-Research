# AIOS 技术分析报告

# AIOS 技术分析报告

## 项目全景概览

**核心用途**: AIOS 是一个将大语言模型嵌入操作系统的 AI Agent 操作系统，旨在为基于 LLM 的智能体提供开发和部署基础设施，解决调度、上下文切换、内存管理、存储管理、工具管理等核心问题。

**核心技术栈**:

- **编程语言**: Python 3.11+, Rust (实验性重写中)
- **Web 框架**: FastAPI, Uvicorn
- **LLM 适配层**: LiteLLM (统一接口支持多个 LLM 提供商)
- **向量数据库**: ChromaDB, Qdrant
- **缓存系统**: Redis
- **配置管理**: YAML, Python-dotenv
- **内存与存储**: Llama Index, Scikit-learn
- **工具集成**: Cerebrum SDK (Agent 开发框架)
- **虚拟化**: Docker, VirtualBox, VMware, Azure, AWS, GCP 云提供商


## 目录结构详解

```
AIOS/
├── aios/                    # 核心 AIOS 内核代码
│   ├── __init__.py         # 包初始化
│   ├── context/            # 上下文管理模块
│   │   ├── base.py         # 上下文管理基类
│   │   └── simple_context.py  # 简单上下文管理器实现
│   ├── config/             # 配置管理模块
│   │   └── config_manager.py  # 单例配置管理器 (YAML 配置)
│   ├── llm_core/           # LLM 核心适配层
│   │   ├── adapter.py      # LLM 适配器 (路由、负载均衡)
│   │   ├── local.py        # 本地 HuggingFace 模型后端
│   │   ├── routing.py      # 路由策略 (顺序、智能路由)
│   │   └── utils.py        # 工具函数 (消息合并、工具调用解码)
│   ├── scheduler/          # 调度器模块
│   │   ├── base.py         # 调度器抽象基类
│   │   ├── fifo_scheduler.py  # FIFO 调度策略
│   │   └── rr_scheduler.py     # 轮询调度策略
│   ├── memory/             # 内存管理模块
│   │   ├── base.py         # 基础内存管理器 (向量检索)
│   │   ├── manager.py      # 内存管理器包装类
│   │   ├── note.py         # 内存笔记数据结构
│   │   └── retrievers.py   # 检索器 (ChromaDB/Qdrant)
│   ├── storage/            # 存储管理模块
│   │   ├── storage.py      # 存储管理器
│   │   └── filesystem/     # 文件系统实现
│   │       ├── lsfs.py     # LSFS 语义文件系统 (向量索引)
│   │       └── vector_db.py # 向量数据库抽象层
│   ├── tool/               # 工具管理模块
│   │   ├── manager.py      # 工具管理器 (MCP Server 集成)
│   │   ├── mcp_server.py   # Model Context Protocol 服务器
│   │   └── virtual_env/    # 虚拟化环境支持
│   │       ├── controllers/    # 控制器 (Python, Setup)
│   │       ├── providers/      # 云提供商
│   │       │   ├── docker/     # Docker 提供商
│   │       │   ├── aws/        # AWS 提供商
│   │       │   ├── azure/      # Azure 提供商
│   │       │   ├── gcp/        # GCP 提供商
│   │       │   └── virtualbox/ # VirtualBox 提供商
│   │       ├── server/      # 服务器实现 (PyxCursor)
│   │       ├── accessibility_tree_wrap/  # 无障碍树包装
│   │       └── evaluators/  # 评估器 (应用特定)
│   ├── syscall/            # 系统调用层
│   │   ├── syscall.py      # 系统调用执行器
│   │   ├── schema.py       # 系统调用模式定义
│   │   ├── factory.py      # 系统调用工厂
│   │   ├── types/          # 系统调用类型定义
│   │   ├── llm.py          # LLM 系统调用
│   │   ├── memory.py       # 内存系统调用
│   │   ├── storage.py      # 存储系统调用
│   │   └── tool.py         # 工具系统调用
│   ├── hooks/              # React Hooks 风格 API
│   │   ├── stores/         # 全局存储 (队列、进程)
│   │   ├── modules/        # 核心模块
│   │   │   ├── llm.py       # useCore LLM 核心
│   │   │   ├── memory.py    # useMemoryManager
│   │   │   ├── storage.py   # useStorageManager
│   │   │   ├── tool.py      # useToolManager
│   │   │   ├── scheduler.py # 调度器 Hooks
│   │   │   └── agent.py     # useFactory Agent 工厂
│   │   ├── types/          # 类型定义
│   │   └── utils/          # 工具函数
│   └── utils/              # 通用工具
│       ├── logger.py       # 日志记录器
│       ├── id_generator.py # ID 生成器
│       ├── calculator.py   # 计算器
│       ├── compressor.py   # 压缩器
│       └── commands/       # 命令工具
│           └── launch.py   # 启动命令
├── aios-rs/                # Rust 重写 (实验性)
│   ├── src/
│   │   ├── lib.rs          # 库入口
│   │   ├── context.rs      # 上下文管理
│   │   ├── llm.rs          # LLM 适配器
│   │   ├── scheduler.rs    # 调度器
│   │   ├── memory.rs       # 内存管理
│   │   ├── storage.rs      # 存储管理
│   │   ├── tool.rs         # 工具管理
│   │   └── prelude.rs      # 预导出模块
│   └── Cargo.toml          # Rust 项目配置
├── runtime/                # 运行时模块
│   ├── launch.py           # FastAPI 主服务器入口
│   └── run_terminal.py     # 终端 UI 运行器
├── scripts/                # 脚本工具
│   ├── run_litellm.py      # LiteLLM 服务器启动脚本
│   ├── run_terminal.py     # 终端 UI 启动脚本
│   └── list_agents.py      # Agent 列出工具
├── tests/                  # 测试模块
│   ├── modules/            # 模块测试
│   │   ├── llm/           # LLM 模块测试
│   │   ├── memory/        # 内存模块测试
│   │   ├── storage/       # 存储模块测试
│   │   ├── tool/          # 工具模块测试
│   │   └── agent_load/    # Agent 加载测试
│   └── test_qdrant_integration.py  # Qdrant 集成测试
├── docs/                   # 文档资源
│   ├── assets/            # 架构图和资源
│   └── CONTRIBUTE.md      # 贡献指南
├── config.yaml             # 主配置文件 (位于 aios/config/)
├── requirements.txt        # Python 依赖
├── Dockerfile             # Docker 镜像定义
└── README.md               # 项目说明文档
```

**核心职责说明**:

- `aios/` - AIOS 内核核心实现，包含所有子系统
- `runtime/launch.py` - FastAPI 服务器，HTTP API 端点
- `aios-rs/` - 性能优化的 Rust 实现（进行中）
- `scripts/` - 辅助脚本和工具
- `tests/` - 各模块单元测试和集成测试


## 核心架构设计

### 设计模式

AIOS 采用了多层架构设计，融合了以下设计模式：

1. **分层架构 (Layered Architecture)**
   - 系统调用层 - 提供统一的系统调用接口
   - Hooks 层 - React Hooks 风格的状态管理和依赖注入
   - 调度器层 - 负责请求调度和资源分配
   - 核心模块层 - LLM、内存、存储、工具管理
   - 基础设施层 - 向量数据库、文件系统、缓存

2. **适配器模式 (Adapter Pattern)**
   - `LLMAdapter` 统一不同 LLM 提供商的接口 (OpenAI, Anthropic, Gemini, Groq, Ollama, HuggingFace, vLLM)
   - LiteLLM 作为底层适配库，提供一致的 API

3. **工厂模式 (Factory Pattern)**
   - `SyscallFactory` 根据查询类型创建对应的系统调用对象
   - `AgentFactory` 根据配置创建不同类型的 Agent 实例

4. **单例模式 (Singleton Pattern)**
   - `ConfigManager` 全局配置管理器
   - 全局请求队列存储 (`global_llm_req_queue_add_message` 等)

5. **策略模式 (Strategy Pattern)**
   - 调度器策略: FIFO、Round Robin
   - LLM 路由策略: Sequential、Smart Routing
   - 向量数据库后端: ChromaDB、Qdrant

6. **观察者模式 (Observer Pattern)**
   - `FileChangeHandler` 监听文件系统变化并更新向量索引
   - Watchdog 文件系统事件监听

7. **代理模式 (Proxy Pattern)**
   - `SyscallExecutor` 代理执行系统调用并收集性能指标

8. **React Hooks 风格**
   - `useCore`, `useMemoryManager`, `useStorageManager` 等 Hooks 提供声明式 API

### 层级调用关系

```
┌─────────────────────────────────────────────────────────┐
│                     Agent / Client                         │
└─────────────────────┬───────────────────────────────────┘
                      │ HTTP / gRPC
┌─────────────────────▼───────────────────────────────────┐
│              FastAPI Server (launch.py)                   │
│           /query endpoint routes to execute_request       │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│         SyscallExecutor (syscall/syscall.py)             │
│    - execute_request() routes queries to syscalls        │
│    - Creates Syscall objects (LLM/Tool/Memory/Storage)   │
│    - Collects timing metrics (waiting, turnaround)       │
└─────────────────────┬───────────────────────────────────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
┌──────────────┐ ┌─────────┐ ┌──────────────┐
│ Global Queues│ │Scheduler│ │  Hooks      │
│ (stores/)    │ │(FIFO/RR)│ │  (modules/) │
└──────┬───────┘ └────┬────┘ └──────┬───────┘
       │              │             │
       ▼              ▼             ▼
┌──────────────┐ ┌─────────┐ ┌──────────────┐
│   Syscalls   │ │   LLM   │ │    Memory    │
│ (llm.py,     │ │ Adapter │ │   Manager    │
│  memory.py,  │ │         │ │              │
│  tool.py,    │ │         │ │              │
│  storage.py) │ │         │ │              │
└──────────────┘ └────┬────┘ └──────┬───────┘
                       │            │
              ┌────────┴────┐  ┌───┴────────┐
              ▼             ▼  ▼             ▼
        ┌─────────┐  ┌─────────┐ ┌─────────┐ ┌─────────┐
        │LiteLLM  │  │  Local  │ │ChromaDB │ │ Qdrant  │
        │OpenAI/  │  │  HF     │ │         │ │         │
        │Anthropic│  │  Models │ │         │ │         │
        └─────────┘  └─────────┘ └─────────┘ └─────────┘

                       │            │            │
              ┌────────┴────────────┴────────────┘
              ▼
        ┌─────────────────┐
        │   Storage        │
        │   Manager        │
        │   (LSFS)         │
        └────────┬─────────┘
                 │
        ┌────────┴────────┐
        ▼                 ▼
   ┌─────────┐      ┌──────────┐
   │ Vector  │      │  Redis   │
   │   DB    │      │  Cache   │
   └─────────┘      └──────────┘
```

**调用流程**:

1. Agent 发送 HTTP 请求到 `/query` 端点
2. `SyscallExecutor.execute_request()` 解析请求类型并创建对应的 `Syscall` 对象
3. Syscall 被添加到全局队列 (LLM/Memory/Storage/Tool)
4. 调度器从队列中取出请求并分发给对应的模块处理
5. 模块调用底层服务 (LiteLLM, Vector DB, Redis, 文件系统)
6. 响应通过 Syscall 返回给 Executor，再由 FastAPI 返回给 Agent


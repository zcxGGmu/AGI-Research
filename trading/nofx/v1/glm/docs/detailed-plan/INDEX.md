# NOFX 港A股重构 - 详细设计文档索引

> **版本**: v3.0 (详细设计版)
> **日期**: 2026-03-19
> **状态**: 编写中

---

## 概述

本文档是 NOFX 港A股 LangGraph 重构项目的详细设计文档索引。基于 v2.0 总体方案，本系列文档提供更细粒度的实现细节，包括完整的代码示例、配置文件和测试用例。

## 文档架构

```
docs/detailed-plan/
├── INDEX.md                    # 本文档 (索引)
├── 01_database_design.md       # 数据库设计
├── 02_api_design.md            # API 接口设计
├── 03_langgraph_nodes.md       # LangGraph 节点设计
├── 04_broker_adapter.md        # 券商适配器设计
├── 05_frontend_components.md   # 前端组件设计
├── 06_test_cases.md            # 测试用例设计
└── 07_deployment.md            # 部署配置设计
```

---

## 文档清单

### 1. 数据库设计 (`01_database_design.md`)

**内容概述**:
- PostgreSQL 配置与连接池策略
- 完整 SQLAlchemy 模型定义
- 索引策略与分区设计
- Alembic 迁移脚本
- 数据约束与安全策略

**关键表**:
| 表名 | 用途 | 分区策略 |
|------|------|----------|
| users | 用户账户 | - |
| traders | 交易机器人配置 | - |
| positions | 持仓信息 | - |
| orders | 订单记录 | 按月分区 |
| trades | 成交记录 | 按月分区 |
| risk_configs | 风险配置 | - |
| langgraph_checkpoints | 工作流检查点 | 按时间分区 |

**状态**: 生成中...

---

### 2. API 接口设计 (`02_api_design.md`)

**内容概述**:
- REST API 设计规范
- WebSocket 实时通信
- JWT 认证流程
- 完整 Pydantic 模型
- 错误处理与限流

**API 分组**:
| 分组 | 前缀 | 功能 |
|------|------|------|
| 认证 | `/api/auth` | 登录、注册、Token 管理 |
| 交易员 | `/api/traders` | Trader CRUD、启停控制 |
| 交易 | `/api/orders` | 下单、撤单、持仓查询 |
| 行情 | `/api/stocks` | 股票搜索、报价、K线 |
| 策略 | `/api/strategies` | 策略配置管理 |
| 工作流 | `/api/workflows` | LangGraph 工作流控制 |

**状态**: 生成中...

---

### 3. LangGraph 节点设计 (`03_langgraph_nodes.md`)

**内容概述**:
- 状态定义 (TradingState 等)
- 节点函数完整实现
- 工具集成 (Tools)
- 图构建与编译
- Checkpointer 配置
- 人机协作实现

**核心节点**:
| 节点 | 功能 | 输入 | 输出 |
|------|------|------|------|
| market_data | 获取行情 | symbol | MarketData |
| research | AI 分析 | MarketData | 分析报告 |
| risk | 风险评估 | 分析报告 | RiskAssessment |
| decide | 决策 | RiskAssessment | 决策结果 |
| execute | 执行交易 | 决策结果 | OrderResult |

**工作流模式**:
- 线性工作流 (简单场景)
- Supervisor 多智能体协作 (复杂场景)
- Human-in-the-loop 审批流程

**状态**: 生成中...

---

### 4. 券商适配器设计 (`04_broker_adapter.md`)

**内容概述**:
- 适配器模式架构
- BrokerAdapter 协议定义
- 富途 OpenD 适配器实现
- 老虎证券适配器实现
- 模拟券商适配器实现
- 错误处理与重连机制

**适配器对比**:
| 功能 | FutuAdapter | TigerAdapter | PaperAdapter |
|------|:-----------:|:------------:|:------------:|
| 港股交易 | ✓ | ✓ | ✓ (模拟) |
| A股交易 | ✓ | ✗ | ✓ (模拟) |
| 实时行情 | ✓ | ✓ | ✓ (模拟) |
| 模拟盘 | ✓ | ✓ | ✓ |
| 回测支持 | ✗ | ✗ | ✓ |

**状态**: 生成中...

---

### 5. 前端组件设计 (`05_frontend_components.md`)

**内容概述**:
- 组件架构概述
- 新增组件完整实现
- 现有组件修改说明
- 自定义 Hooks
- API 客户端封装
- 状态管理策略

**新增组件**:
| 组件 | 用途 | 关键功能 |
|------|------|----------|
| StockSearch | 股票搜索 | 搜索、筛选、市场标识 |
| StockQuote | 股票报价 | 实时价格、涨跌幅 |
| MarketSelector | 市场选择 | 港股/A股切换 |
| TradingHoursBadge | 交易时段 | 开盘/收盘状态 |
| StockOrderForm | 下单表单 | 数量、价格、类型 |
| WorkflowApproval | 审批组件 | 人机协作审批 UI |

**状态**: 生成中...

---

### 6. 测试用例设计 (`06_test_cases.md`)

**内容概述**:
- 测试策略概述
- 单元测试完整用例
- 集成测试完整用例
- E2E 测试用例
- 测试 Fixtures
- Mock 策略

**测试覆盖率目标**:
| 类型 | 占比 | 覆盖率目标 |
|------|------|-----------|
| 单元测试 | 60% | 85%+ |
| 集成测试 | 30% | 80%+ |
| E2E 测试 | 10% | 关键路径 100% |

**关键测试场景**:
- 交易时段判断
- 涨跌停检测
- 风险计算
- 订单验证
- 工作流审批

**状态**: 生成中...

---

### 7. 部署配置设计 (`07_deployment.md`)

**内容概述**:
- 部署架构概述
- Docker 多阶段构建
- Kubernetes 完整配置
- CI/CD Pipeline
- 监控告警配置
- 运维手册

**部署架构**:
```
┌─────────────────────────────────────────┐
│              Kubernetes 集群             │
├─────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
│  │   API   │  │ Worker  │  │ OpenD   │  │
│  │ (3 pods)│  │ (2 pods)│  │ (1 pod) │  │
│  └────┬────┘  └────┬────┘  └────┬────┘  │
│       │            │            │       │
│  ┌────┴────────────┴────────────┴────┐  │
│  │           PostgreSQL              │  │
│  │         (Primary + Replica)       │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

**状态**: 生成中...

---

## 使用指南

### 文档阅读顺序

1. **架构理解**: 先阅读 `NOFX_HK_ASHARE_LANGGRAPH_REFACTOR_PLAN.md` 总体方案
2. **数据库设计**: 理解数据模型，为后续开发打基础
3. **API 设计**: 了解前后端接口契约
4. **LangGraph 节点**: 理解核心业务逻辑
5. **券商适配器**: 了解外部系统集成
6. **前端组件**: 前端开发参考
7. **测试用例**: 确保代码质量
8. **部署配置**: 生产环境部署

### 开发启动检查清单

- [ ] 阅读总体方案 (v2.0)
- [ ] 理解数据库模型
- [ ] 熟悉 API 接口设计
- [ ] 了解 LangGraph 工作流
- [ ] 配置本地开发环境
- [ ] 运行测试验证环境

---

## 技术栈速查

### 后端 (Python)

```toml
[tool.poetry.dependencies]
python = "^3.11"
fastapi = "^0.115.0"
langgraph = "^0.2.0"
langchain-openai = "^0.2.0"
sqlalchemy = "^2.0"
asyncpg = "^0.29"
alembic = "^1.13"
futu-api = "^6.5.0"
akshare = "^1.14"
pydantic = "^2.0"
```

### 前端 (TypeScript)

```json
{
  "dependencies": {
    "react": "^18.0",
    "typescript": "^5.0",
    "tailwindcss": "^3.0",
    "@tanstack/react-query": "^5.0"
  }
}
```

### 基础设施

- **容器**: Docker + docker-compose
- **编排**: Kubernetes (生产)
- **数据库**: PostgreSQL 15+
- **缓存**: Redis 7+
- **监控**: Prometheus + Grafana + Loki

---

## 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| v1.0 | 2026-03-19 | 初始总体方案 |
| v2.0 | 2026-03-19 | 增加 Mermaid 图表、完善测试部署章节 |
| v3.0 | 2026-03-19 | 详细设计文档系列，提供完整代码示例 |

---

> **文档维护**: 本文档系列随项目进展持续更新。如有疑问，请参考主项目文档或联系开发团队。
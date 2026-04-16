# 元认知技能体系

一个三层技能生成系统，支持跨领域的系统性方法论构建和技能创建。

## 概述

本项目实现了一种元认知方法来生成技能，具有三个明确的层次：

| 层级 | 名称 | 描述 | 位置 |
|------|------|------|------|
| **L1** | 元元技能 | 顶层认知原则与方法论生成指南 | `meta-skills/` |
| **L2** | 元技能 | 特定领域的方法论，用于生成L3技能 | `examples/web-frontend-skill-creator/` |
| **L3** | 普通技能 | 具体场景的执行技能 | `examples/web-frontend-skills/` |

## 架构

```
skill-master/
├── SKILL.md                           # L1导航 (meta-cognitive-skill-system)
├── meta-skills/                       # L1: 元元技能 (6个静态文件)
│   ├── epistemology.md                # mm-001: 认识论框架
│   ├── decomposition.md               # mm-002: 分解与抽象
│   ├── causal-thinking.md             # mm-003: 因果与系统思维
│   ├── meta-cognition.md             # mm-004: 元认知与自我反思
│   ├── value-priority.md            # mm-005: 价值与优先级框架
│   └── interaction.md               # mm-006: 交互与沟通元模型
├── examples/
│   ├── web-frontend-skill-creator/   # L2: Web前端方法论 (由L1生成)
│   │   └── SKILL.md
│   └── web-frontend-skills/          # L3: 具体前端技能 (由L2生成)
│       ├── form-handling/
│       ├── state-management-implementation/
│       ├── routing-and-navigation/
│       └── ... (20+个技能)
└── CLAUDE.md                          # 项目指南
```

## 各层职责

### L1: 元元技能（静态）

L1元元技能**不直接参与日常任务**。它们只在生成或修订L2方法论时被调用。

| 技能 | 作用 |
|------|------|
| **认识论框架** | 定义"怎么判断对错" —— 建立领域的认知标准 |
| **分解与抽象** | 定义"怎么拆解问题" —— 建立领域的结构视图 |
| **因果与系统思维** | 定义"怎么理解因果" —— 建立领域的动态模型 |
| **元认知与自我反思** | 定义"怎么审视自己" —— 建立质量检查机制 |
| **价值与优先级框架** | 定义"什么更重要" —— 建立决策排序标准 |
| **交互与沟通元模型** | 定义"怎么与人交互" —— 确保方法论的用户接口质量 |

### L2: 元技能（动态）

L2元技能是**特定领域的方法论**，回答"在这个领域应该怎么思考和行动"。它们**只能用于生成L3技能**，不能直接解决问题。

示例：[web-frontend-methodology](examples/web-frontend-skill-creator/SKILL.md)

### L3: 普通技能（动态）

L3技能是**具体场景的执行技能**，回答"在这个场景下具体怎么做"。

示例：[form-handling](examples/web-frontend-skills/form-handling/SKILL.md)

## 生成流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    生成L2方法论                                  │
├─────────────────────────────────────────────────────────────────┤
│  1. 加载L1元元技能                                               │
│     │                                                            │
│     ├──→ [串行] 认识论框架 → 权威资料搜索                         │
│     ├──→ [串行] 分解与抽象                                       │
│     ├──→ [串行] 因果与系统思维                                    │
│     ├──→ [串行] 元认知与自我反思                                  │
│     ├──→ [串行] 价值与优先级框架                                  │
│     └──→ [并行] 交互与沟通元模型                                  │
│                                                                  │
│  2. 整合生成L2方法论（必须符合L2输出格式规范）                    │
│                                                                  │
│  3. 质量审核                                                      │
│                                                                  │
│  4. 输出为SKILL.md到L2存放目录                                   │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                    生成L3普通技能                                 │
├─────────────────────────────────────────────────────────────────┤
│  基于已验证的L2:                                                  │
│  1. 确定具体场景和任务                                           │
│  2. 从L2中提取相关原则                                           │
│  3. 添加具体步骤、示例、注意事项                                  │
│  4. 输出到L3存放目录                                             │
└─────────────────────────────────────────────────────────────────┘
```

## 示例技能（L3）

`web-frontend-skills/` 目录包含20+个使用Web前端方法论生成的具体技能：

- [form-handling](examples/web-frontend-skills/form-handling/SKILL.md) — 表单状态管理、验证、提交
- [state-management-implementation](examples/web-frontend-skills/state-management-implementation/SKILL.md) — 状态管理模式
- [routing-and-navigation](examples/web-frontend-skills/routing-and-navigation/SKILL.md) — 客户端路由
- [performance-optimization](examples/web-frontend-skills/performance-optimization/SKILL.md) — 性能最佳实践
- [error-boundary-and-fallback](examples/web-frontend-skills/error-boundary-and-fallback/SKILL.md) — 错误处理
- [api-data-fetching-and-caching](examples/web-frontend-skills/api-data-fetching-and-caching/SKILL.md) — 数据获取模式
- [lazy-loading-and-code-splitting](examples/web-frontend-skills/lazy-loading-and-code-splitting/SKILL.md) — 代码分割
- [automated-testing-strategy](examples/web-frontend-skills/automated-testing-strategy/SKILL.md) — 测试方法
- [access-control-implementation](examples/web-frontend-skills/access-control-implementation/SKILL.md) — 认证与授权
- [chart-visualization-integration](examples/web-frontend-skills/chart-visualization-integration/SKILL.md) — 数据可视化
- ...以及更多

## 设计原则

1. **渐进式披露** — 技能按需加载上下文：元数据（~100 tokens）→ 核心指令（~500 tokens）→ 详细参考（按需）

2. **L1不可变原则** — L1元元技能一旦发布，内容不再修改。新版本与旧版本共存。

3. **职责分离** — L2只生成L3；L3执行具体任务。直接解决问题始终由L3完成。

4. **权威知识优先** — L2生成前必须搜索权威资料（学术论文、行业标准、专家观点），建立认知标准。

## 使用说明

本节介绍如何在 Agent 工具（如 Claude Code 或 OpenClaw）中使用本系统，包括如何编写提示词来生成领域元技能（L2）和具体技能（L3）。

### 系统工作原理

当你激活 `meta-cognitive-skill-system` 技能（通过根目录 `SKILL.md`）时，系统会：
1. 加载相应的 L1 元元技能
2. 根据你的提示词引导生成过程
3. 输出结构化的 SKILL.md 文件

### 编写生成 L2 的提示词

当你需要在一个**新领域**工作且没有现成的 L2 方法论时：

**提示词模板：**
```
我需要在 [领域名称] 工作。请帮助我生成该领域的 L2 方法论。

领域描述：[简要描述该领域]

请执行以下操作：
1. 激活 meta-cognitive-skill-system 技能
2. 执行 L2 生成工作流
3. 搜索该领域的权威资料
4. 生成完整的 L2 方法论 SKILL.md
```

**示例提示词：**
- `"我需要进行 Go 后端服务开发。请为后端开发领域生成一个 L2 方法论。"`
- `"帮助我创建一个数据工程的方法论。我需要系统化的数据管道构建方法。"`
- `"我们准备进入移动开发领域。请为 iOS 开发生成一个 L2 方法论。"`

### 编写生成 L3 的提示词

当你已有 L2 方法论，需要某个**具体场景**的技能时：

**提示词模板：**
```
我需要在 [领域] 中 [具体任务]。请使用 [领域] 方法论（L2）
为这个场景生成一个 L3 技能。

具体任务是：[详细描述任务]

请执行以下操作：
1. 加载 [领域]-methodology 技能
2. 提取与该场景相关的原则
3. 生成具体的 L3 技能 SKILL.md
```

**示例提示词：**
- `"我需要在 React 应用中实现 JWT 认证。请使用 web-frontend 方法论生成一个 JWT 认证的 L3 技能。"`
- `"我们正在构建带错误处理的 REST API。请使用后端方法论生成一个 API 错误处理的 L3 技能。"`
- `"我需要处理带进度显示的文件上传。请生成一个 L3 技能。"`

### 编写修订 L2 的提示词

当现有 L2 方法论存在问题需要修订时：

**提示词模板：**
```
我发现 [领域] 方法论有问题。[描述问题]。

具体问题是：[出了什么问题]
涉及维度可能是：[认识论 / 分解与抽象 / 因果思维 / 元认知 / 价值优先级 / 交互沟通]

请执行以下操作：
1. 激活 meta-cognitive-skill-system 技能
2. 定位相关的 L1 技能
3. 只修订受影响的部分
4. 更新方法论版本
```

### 激活条件速查

| 你的情况 | 正确的提示词风格 | 错误的提示词风格 |
|----------|------------------|------------------|
| "需要进入后端开发领域" | `"为后端开发生成一个 L2 方法论"` | `"帮我设计这个数据库表结构"` |
| "需要一个表单技能" | `"使用 web-frontend 方法论生成一个表单处理的 L3 技能"` | `"React 表单验证怎么做？"` |
| "设计一个登录表单" | `"使用 form-handling L3 技能来指导实现"` | `"为前端生成 L2"` |

### 提示词关键词参考

| 意图 | 推荐关键词 |
|------|-----------|
| 生成 L2 | `"generate L2 methodology"`, `"create domain methodology"`, `"为 [领域] 生成方法论"` |
| 生成 L3 | `"generate L3 skill"`, `"create [场景] skill"`, `"使用 [L2名称] 生成技能"` |
| 修订 L2 | `"revise methodology"`, `"update L2"`, `"方法论在 [方面] 有问题"` |
| 进入新领域 | `"new domain"`, `"first time working in"`, `"在没有现成方法论的领域工作"` |

## 快速参考

| 动作 | 触发条件 |
|------|----------|
| 进入全新领域 | 运行完整的L2生成流程，调用全部L1 |
| 现有L2部分失效 | 定位问题，只调用相关的L1 |
| 生成L3技能 | 使用L2方法论生成具体技能 |

## 许可证

MIT

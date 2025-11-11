# AI Agents

## 基础理论与关键概念

### 如何定义 AI Agent？与传统的聊天机器人或自动化脚本的核心区别是什么？

https://mp.weixin.qq.com/s/tewBKHgbyrjxUjAOmkXI7A

### 描述 ReAct、CoT、Reflection、Tree of Thoughts 的原理与适用场景。

### 什么是工具调用（Tool Use / Function Calling）？如何设计一个安全且稳定的工具协议？

### 大模型在 Agent 场景中常见的幻觉来源是什么？有哪些缓解手段？

### 解释 LangChain、LlamaIndex 与自研 Agent 框架的核心差异。

## 模型与推理调优

### 在 Agent 系统中，如何设计 Prompt 模块化？如何构建可组合的 Prompt Chain？

### 如何让模型保持长期任务上下文？你用过哪些 Memory 机制？

### 在 Agent 中如何评价模型“推理能力”与“决策能力”？你会设计哪些指标？

### 如何在多工具、多步骤推理场景中控制 token 成本？

### 你是否做过 ReAct 的裁剪或增强？举例说明优化策略和效果。

## 工具集成与动作执行（Tool/Action Layer）

### 你在实际项目中集成过哪些工具？（搜索、数据库、CRM、代码执行等）

### 如何设计一个“通用工具协议”，让模型能自动适配未来新增工具？

### Tool Calling 出现“参数错误 / 无限循环调用 / 工具调用中断”时，你如何检测和恢复？

### 你如何实现 Agent 的副作用安全（Side-effect Safety）？

### 如何自动从 API 文档生成可调用的 Function Schema？

## Agent 架构设计与系统工程

### 请设计一个企业级多代理协作系统（如 QA + Planner + Executor）。你如何处理：

• 角色分工
• 中间状态管理
• 调度与仲裁
• 失败恢复

### 如何在高并发下运行成千上万个任务型 Agent？

### Agent 的“长任务”（Long-running Tasks）如何保持状态一致性？

### 如何设计一个可插拔式 Planner，使 Agent 可根据任务自动选择策略（如 BFS/DFS 规划、工具优先、模型优先）？

### 如何为 Agent 构建一个可观察性（Observability）系统？需要监控哪些指标？

## 数据与知识集成（Knowledge & Retrieval）

### 如何设计企业级检索增强（RAG）系统？你如何避免检索噪声？

### 如何在 Agent 中结合结构化数据与非结构化数据？

### 面对用户上传的文档，如何自动生成文档工具调用方案？

### Hybrid RAG、Graph RAG、Agentic RAG 的差异？

### 如何追踪每一步推理的知识来源？

## 鲁棒性、安全性与可信执行

### 如何检测 Agent 是否进入循环？你的终止条件是什么？

### 对于“模型产生错误命令”，你的保护机制是什么？

### 你如何防御用户 prompt injection？

### 工具返回异常数据时，你如何自动重试或切换策略？

### 如何实现可审计（Auditable）的 Agent 推理日志？

## 评估（Evaluation）

### 如何对任务型 Agent 进行系统性评估？有哪些维度？

### 你会如何搭建 E2E agent benchmark，包括：

• 覆盖任务集
• 自动可重复执行
• 成功判定

### 如何评估工具调用的准确率与鲁棒性？

### 如何评估 Agent 的“可控性”？

### 如何测试模型是否会在工具调用中产生幻觉 Schema？

## 工程实践与业务落地

### 分享你做过的最复杂的 AI Agent 项目，遇到的最大技术难题是什么？

### 你如何推动 Agent 从 Demo 到生产环境（Productionization）？

### 如何与业务团队合作拆解需求为 Agent 任务？

### 如何控制上线后的模型成本与质量？

### 请举例说明一次你在真实场景中大幅提升 Agent 成功率的优化经验。

## 课程

https://huggingface.co/learn/agents-course

# 课题任务书

## 方向：软件开发项目全生命周期管理Agent

---

### 一、项目背景

软件开发团队（初创公司研发团队、企业IT部门、外包团队等）在项目管理中普遍面临三类问题：其一，任务分解、计划排期、进度跟踪依赖项目经理手动维护，信息容易滞后或失真；其二，缺陷（Bug）跟踪与代码审查（Code Review）提醒缺乏自动化机制，团队成员易遗漏关键事项；其三，版本发布、测试用例管理等重复性流程占用研发人员大量精力。

针对上述问题，本课题研究方向为：构建一个面向软件开发团队的项目全生命周期管理智能助手，能够辅助完成需求拆解、任务派发、Bug跟踪提醒、版本发布跟进、测试用例管理等工作，提升研发协作效率与项目状态可见性。

---

### 二、研究思路

本课题的整体研究思路遵循「问题分析—方案设计—系统实现—测试验证」的递进逻辑，分四个阶段展开。

第一阶段：问题分析。通过文献调研梳理软件项目管理方法论与智能研发辅助工具的研究现状；通过研发团队调研，识别项目管理中的重复劳动与协作痛点。

第二阶段：方案设计。研究将围绕「项目状态的智能跟踪」「缺陷与代码审查的自动化提醒」「版本与测试的流程辅助」三个核心问题进行方案设计。在项目状态跟踪方面，研究如何从代码托管平台（如Git）、即时通讯、项目看板等多源数据自动汇总项目状态；在Bug与代码审查提醒方面，研究如何基于规则与活动情况自动识别待处理事项并推送提醒；在版本与测试辅助方面，研究如何辅助版本发布、测试用例生成与管理。

第三阶段：系统实现。搭建包含项目管理数据接入、智能分析、自动化提醒、流程辅助模块的系统原型，对接至少一个代码托管平台与即时通讯平台。

第四阶段：测试验证。通过在真实项目（含开源项目或学生团队项目）中的试用，验证系统在提升项目透明度、减少重复工作、加速协作方面的有效性。

---

### 三、技术选型方向

**1. 智能体技术选型**。从以下主流框架中选定：
   - **LangChain/LangGraph**
   - **AutoGen**
   - **CrewAI**
   - **AgentScope**

**2. 大语言模型接入模式**。从以下两种模式中选定一种：
   - **API调用模式**：通过云服务API访问大语言模型；
   - **本地部署模式**：在本地或私有服务器部署开源模型（建议RTX 3090以上显卡，部署10B以下规模模型；70B级模型需多卡服务器）。

**3. 领域关键技术**。本课题涉及以下关键技术的选型：
   - **Git平台API**：GitHub/Gitee/GitLab开放API
   - **即时通讯Webhook**：企业微信/钉钉/飞书机器人
   - **定时任务**：Celery Beat（日报/周报汇总）
   - **向量数据库**：Chroma（代码变更摘要、Issue聚类）
   - **关系型数据库**：PostgreSQL（支持JSON字段，存储Git事件原始数据）

### 四、需要完成的工作

1. 文献与产品调研：完成软件项目管理工具与智能研发辅助产品调研报告；
2. 需求分析：完成目标研发团队画像分析，输出系统用例图与功能需求规格说明书；
3. 技术方案设计：通过技术对比确定所采用的技术栈与系统架构，输出系统设计说明书；
4. 核心功能开发：实现项目数据接入与汇总、智能任务提醒、Bug/代码审查跟踪、版本与测试辅助等核心功能；
5. 数据层搭建：构建项目状态模型、Bug与代码审查跟踪模型、版本与测试用例管理模型；
6. 系统集成与测试：完成系统集成，在真实项目中试用并输出测试报告；
7. 论文撰写：按学校规范完成毕业论文（约1.5-2万字），包括中英文摘要、目录、正文及参考文献等部分。

---

### 五、参考文献

[1] Anthropic. Building Effective Agents[EB/OL]. (2024-12-19). https://www.anthropic.com/engineering/building-effective-agents.

[2] Wei J, Wang X, Schuurmans D, et al. Chain-of-Thought Prompting Elicits Reasoning in Large Language Models[C]// Advances in Neural Information Processing Systems 35. 2022: 24824-24837.

[3] Cheng Y, Zhang C, Zhang Z, et al. Exploring Large Language Model based Intelligent Agents: Definitions, Methods, and Prospects[J]. arXiv:2401.03428, 2024.

[4] Yao S, Zhao J, Yu D, et al. ReAct: Synergizing Reasoning and Acting in Language Models[C]// International Conference on Learning Representations (ICLR). 2023.

[5] Schick T, Dwivedi-Yu J, Dessì R, et al. Toolformer: Language Models Can Teach Themselves to Use Tools[C]// Advances in Neural Information Processing Systems 36. 2023.

[6] Anthropic. How We Built Our Multi-Agent Research System[EB/OL]. (2025-04). https://www.anthropic.com/engineering/multi-agent-research-system.
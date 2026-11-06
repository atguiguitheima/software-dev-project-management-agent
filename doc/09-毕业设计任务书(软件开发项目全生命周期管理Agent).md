# 广东工业大学本科生毕业设计（论文）任务书

| 项 目     | 内 容                             |
| ------- | ------------------------------- |
| 题 目 名 称 | 基于大语言模型智能体的软件开发项目全生命周期管理助手设计与实现 |
| 学　　　院   | 管理学院                            |
| 专 业 班 级 | 信息管理与信息系统 23-卓越班                |
| 姓　　　名   | 黄彬                              |
| 学　　　号   | 3123003144                      |

---

## 一、毕业设计（论文）的内容与要求

### 1.1 研究背景与问题

软件开发团队（初创公司研发团队、企业 IT 部门、外包团队等）在项目管理中普遍面临三类问题：其一，任务分解、计划排期、进度跟踪依赖项目经理手动维护，信息容易滞后或失真；其二，缺陷（Bug）跟踪与代码审查（Code Review）提醒缺乏自动化机制，团队成员易遗漏关键事项；其三，版本发布、测试用例管理等重复性流程占用研发人员大量精力。

上述问题的结构性根源在于：**项目事实发生在代码托管平台，而人的注意力停留在即时通讯工具中**。事实与注意力之间存在天然鸿沟，导致项目状态可见性不足、协作延迟、重复劳动难以消除。

### 1.2 研究目标

构建一个面向软件开发团队的项目全生命周期管理智能助手，以「事实采集—状态投影—规则筛选—智能处理—触达执行」为主线，自动完成多源项目数据接入、领域状态建模、待办事项识别与提醒、需求拆解与任务派发、版本发布跟进与测试用例管理，并通过真实项目试用验证系统在提升项目透明度、减少重复工作、加速协作方面的有效性。

### 1.3 具体内容

**（1）多源数据接入层**

- **GitHub 侧**：采用「Webhook 主通道 + REST API 轮询对账」双通道。Webhook 负责实时接收 push、pull_request、pull_request_review、issues、issue_comment、release、check_run 等事件，须完成 HMAC-SHA256 验签、投递 ID 幂等去重、原始 payload 原样落库；轮询通道通过对快照做差分还原事件，兜住 Webhook 漏投与服务停机期间的数据。采集内容包括 PR 与审查人/审查记录、Issues 与评论、Commits、Releases、协作者、CI 运行状态。
- **飞书侧**：采用自建应用 + 长连接（WebSocket）事件订阅接收消息与卡片回调，通过 `im/v1/messages` REST 接口发送消息与交互卡片，实现双向对话与可操作提醒。
- **统一事件模型**：接入层不直接处理业务，而是将外部协议翻译为内部领域事件，使处理器与具体平台解耦。

**（2）数据处理流水线**

设计五段主流水线（摄入 → 规范化 → 状态投影 → 规则筛选 → LLM 处理 → 动作输出）与一条向量化旁路。核心原则为**漏斗式收敛**：越昂贵的处理（LLM）位置越靠后、输入规模越小。全链路须保证幂等（可安全重试）、可观测（运行记录与处理标记落库）、可重放（原始事件保留，状态可重建）。处理单元按聚合（如仓库+PR 编号）划分，同一聚合并发串行、不同聚合并行。

**（3）存储层**

- **MySQL 8.0 + GORM**：存储事件原始数据（JSON 列）、项目与任务状态、Bug 与代码审查跟踪模型、版本与测试用例模型、提醒去重表、Outbox 表；字符集统一 utf8mb4。
- **Milvus**：存储代码变更摘要与 Issue 的向量表示，支撑相似检索与聚类。
- **双库一致性**：MySQL 为唯一事实源，Milvus 为可重建的派生索引；通过 Transactional Outbox（业务行与待嵌入任务在同一事务落库）+ 按 source_id 幂等 upsert + 定时对账实现最终一致。
- **Redis（可选）**：用于 LLM 响应缓存与推送/调用限流，不做事实源。

**（4）Agent 核心执行引擎**

- 实现 ReAct 循环（Thought / Action / Observation）与工具注册表，支持 Function Calling；
- 采用**双实现路线**：自研循环与 Eino 框架实现，统一 `Runner` 接口，通过配置切换；工具业务逻辑只写一份，两侧各包一层适配；
- 记忆机制：短期上下文（截断与摘要）+ 长期领域库（工具查询）+ 向量检索；
- 护栏设计：最大步数、工具超时降级、结果截断、写类工具需确认。

**（5）业务功能**

- 项目状态日报：定时聚合多源状态，一次归纳后推送飞书卡片；
- Code Review 超时提醒：规则先筛候选，再生成个性化提醒，按「每人每事每天一次」去重；
- Bug 跟踪提醒：严重级判定、响应监测，并附带历史相似问题；
- 需求拆解与任务派发：对话式生成任务树，先落草稿、确认后生效，可回写 GitHub Issue；
- 版本发布跟进与测试用例管理：发布检查单、用例草稿生成与状态流转。

**（6）评估与验证**

在真实项目（开源项目或学生团队项目）中试用，统计并分析提醒准确率与召回率、PR/Issue 响应时长变化、工具调用成功率、端到端延迟与 token 消耗；并开展自研引擎与 Eino 引擎的对照实验。

### 1.4 要求

**1. 技术栈要求**

- 开发语言：Go 1.23（单二进制部署，并发模型适合事件处理）；
- 数据访问：GORM（MySQL 驱动 go-sql-driver/mysql），遵循既定使用规范；
- 关系数据库：MySQL 8.0（utf8mb4 + JSON 类型）；
- 向量数据库：Milvus standalone（milvus-sdk-go）；
- 定时任务：robfig/cron v3；
- Web 框架：chi（Webhook 接收与管理 API）；
- 平台对接：GitHub（google/go-github，鉴权 PAT 起步、演进至 GitHub App）、飞书（larksuite/oapi-sdk-go v3）；
- 大语言模型：采用 API 调用模式，兼容 OpenAI 接口协议；
- 缓存：Redis（按需引入，仅用于缓存与限流）。

**2. 工程要求**

- 外部写操作（发消息、回写 GitHub）一律经持久化意图 + 幂等重试 + 对账，禁止业务代码直接双写；
- 所有外部事件与消息须按唯一键去重，保证至少一次投递下的幂等；
- 时区在接入层统一转换为北京时间后再落库；
- 每次 LLM 运行的 token、耗时、工具轨迹须落库，支撑评估与问题定位；
- 单元测试须覆盖 Webhook 验签、事件差分、状态机转移、提醒去重等关键逻辑。

**3. 对接要求**：至少完成一个代码托管平台（GitHub）与一个即时通讯平台（飞书）的实际对接，并提供可演示的完整链路。

**4. 交付物要求**

- 可运行的系统源码与使用说明；
- 数据库迁移脚本与系统部署配置；
- 需求规格说明书、系统设计说明书；
- 真实项目试用测试报告；
- 毕业论文：约 2 万字，包含中英文摘要、目录、正文及参考文献等部分；查阅相关重要文献 20 篇以上，参考文献不少于 15 篇，其中外文文献不少于 2 篇。

**5. 过程要求**：认真领会任务书规定任务，在教师指导下制定进程安排；严格遵守纪律，按进程安排执行；按学校格式规范撰写论文，保证按期、按质、按量完成；坚持科学态度，遵守学术道德，不弄虚作假、不抄袭他人成果；做好进展记录与阶段总结，定期向指导教师汇报（每周至少 1 次）；论文数据应权威准确、及时新颖，观点及理论应详细具体、逻辑严密。

---

## 二、毕业设计（论文）应完成的工作

1. **调研与综述**：调研软件项目管理工具与智能研发辅助产品现状，梳理大语言模型智能体（规划、记忆、工具使用、反思）与软件工程智能化（LLM4SE）的研究进展，输出调研报告；
2. **需求分析**：完成目标研发团队画像分析，明确用户角色与场景，输出系统用例图与功能需求规格说明书；
3. **技术方案设计**：对候选框架、数据库、接入方案做技术对比，确定技术栈与分层架构，输出系统设计说明书（含状态机、表结构、接口定义）；
4. **数据层搭建**：完成 MySQL 迁移脚本与 GORM 模型定义、Milvus collection 设计、Outbox 与提醒去重表设计；
5. **多源接入实现**：实现 GitHub Webhook 接收（验签/幂等/落库）与轮询差分采集；实现飞书长连接收发与卡片交互；完成账号映射；
6. **核心引擎与业务实现**：实现 ReAct 循环与工具集（GitHub 只读、飞书写、领域读写、RAG 四组），实现五类业务工作流与定时调度；
7. **集成验证与论文撰写**：完成系统集成与真实项目试用，输出测试报告；按学校规范撰写并完成毕业论文，做好答辩与归档。

---

## 三、毕业设计（论文）进程安排

| 序号 | 设计（论文）各阶段内容 | 起止日期                    |
| -- | ----------- | ----------------------- |
| 1  | 教师拟题        | 2026-11-15 至 2026-11-17 |
| 2  | 导师确认学生      | 2026-11-23 至 2026-11-24 |
| 3  | 毕业论文动员会     | 2026-11-25 至 2026-11-25 |
| 4  | 教师拟定任务书     | 2026-11-26 至 2026-11-30 |
| 5  | 学生确定提纲      | 2026-12-01 至 2026-12-14 |
| 6  | 学生撰写论文初稿    | 2026-12-15 至 2027-03-10 |
| 7  | 论文中期检查      | 2027-03-10 至 2027-03-15 |
| 8  | 论文二稿/三稿及修改  | 2027-03-15 至 2027-05-06 |
| 9  | 毕业论文终稿提交    | 2027-05-07 至 2027-05-08 |
| 10 | 论文答辩        | 2027-05-20 至 2027-06-01 |
| 11 | 论文答辩后修改     | 2027-06-02 至 2027-06-08 |
| 12 | 论文相关资料整理归档  | 2027-06-08 至 2027-06-18 |

---

## 四、应收集的资料及主要参考文献

### 4.1 应收集的主要资料

1. 目标项目在 GitHub 上的仓库数据：Pull Request（含审查人与审查记录）、Issues 与评论、Commits、Releases、协作者名单、CI 运行状态；
2. 飞书开放平台接口文档：事件订阅（长连接）、消息发送、交互卡片与卡片回调；
3. 系统运行数据：每次 LLM 运行的输入输出、工具调用轨迹、token 消耗与耗时；
4. 试用反馈数据：提醒的准确率与召回率统计、PR/Issue 响应时长的前后对比、用户问卷与访谈记录。

### 4.2 主要参考文献

**智能体基础理论与方法**

[1] Anthropic. Building Effective Agents[EB/OL]. (2024-12-19)[2026-10-30]. <https://www.anthropic.com/engineering/building-effective-agents>.

[2] Yao S, Zhao J, Yu D, et al. ReAct: Synergizing Reasoning and Acting in Language Models[C]//Proceedings of the Eleventh International Conference on Learning Representations (ICLR). 2023.

[3] Schick T, Dwivedi-Yu J, Dessì R, et al. Toolformer: Language Models Can Teach Themselves to Use Tools[C]//Advances in Neural Information Processing Systems 36 (NeurIPS 2023). 2023.

[4] Wei J, Wang X, Schuurmans D, et al. Chain-of-Thought Prompting Elicits Reasoning in Large Language Models[C]//Advances in Neural Information Processing Systems 35 (NeurIPS 2022). 2022: 24824-24837.

[5] Wang L, Ma C, Feng X, et al. A Survey on Large Language Model Based Autonomous Agents[J]. Frontiers of Computer Science, 2024, 18(6): 186345. DOI: 10.1007/s11704-024-40231-1.

[6] Cheng Y, Zhang C, Zhang Z, et al. Exploring Large Language Model Based Intelligent Agents: Definitions, Methods, and Prospects[J]. arXiv:2401.03428, 2024.

**多智能体与软件工程应用**

[7] Anthropic. How We Built Our Multi-Agent Research System[EB/OL]. (2025-04)[2026-10-30]. <https://www.anthropic.com/engineering/multi-agent-research-system>.

[8] Wu Q, Bansal G, Zhang J, et al. AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation[C]//arXiv:2308.08155, 2023.

[9] Hong S, Zheng X, Chen J, et al. MetaGPT: Meta Programming for a Multi-Agent Collaborative Framework[C]//Proceedings of the Twelfth International Conference on Learning Representations (ICLR). 2024.

[10] Hou X, Zhao Y, Liu Y, et al. Large Language Models for Software Engineering: A Systematic Literature Review[J]. ACM Transactions on Software Engineering and Methodology, 2024, 33(8): 1-79. DOI: 10.1145/3695988.

[11] Jimenez C E, Yang J, Wettig A, et al. SWE-bench: Can Language Models Resolve Real-World GitHub Issues?[C]//Proceedings of the Twelfth International Conference on Learning Representations (ICLR). 2024.

[12] 董益宏, 姜雪, 钱家如, 王天, 张克驰, 金芝, 李戈. 基于大语言模型智能体的代码生成综述[J]. 软件学报, 2026, 37(8): 3089-3115. DOI: 10.13328/j.cnki.jos.007593.

[13] 康俊驰, 丁博, 冯大为, 翟远钊, 张迅晖, 王怀民. 大语言模型智能体在软件系统根因分析中的应用综述[J]. 软件学报. DOI: 10.13328/j.cnki.jos.007594.

**项目管理与研发协作自动化**

[14] Cinkusz K, Chudziak J A, Niewiadomska-Szynkiewicz E. Cognitive Agents Powered by Large Language Models for Agile Software Project Management[J]. Electronics, 2024, 14(1): 87. DOI: 10.3390/electronics14010087.

[15] Maddila C, Upadrasta S S, Bansal C, et al. Nudge: Accelerating Overdue Pull Requests Towards Completion[J]. ACM Transactions on Software Engineering and Methodology, 2022. DOI: 10.1145/3544791.

[16] Hasan K A, Macedo M, Tian Y, et al. Understanding the Time to First Response in GitHub Pull Requests[C]//Proceedings of the 20th International Conference on Mining Software Repositories (MSR). 2023.

[17] Peng S, Kalliamvakou E, Cihon P, et al. The Impact of AI on Developer Productivity: Evidence from GitHub Copilot[J]. arXiv:2302.06590, 2023.

**检索增强与数据基础设施**

[18] Lewis P, Perez E, Piktus A, et al. Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks[C]//Advances in Neural Information Processing Systems 33 (NeurIPS 2020). 2020: 9459-9474.

[19] Wang J, Yi X, Guo R, et al. Milvus: A Purpose-Built Vector Data Management System[C]//Proceedings of the 2021 International Conference on Management of Data (SIGMOD '21). New York: ACM, 2021: 2614-2627. DOI: 10.1145/3448016.3457550.

[20] Kleppmann M. Designing Data-Intensive Applications[M]. Sebastopol: O'Reilly Media, 2017.（中译本：数据密集型应用系统设计[M]. 赵军平, 吕云松, 耿煜, 等译. 北京: 中国电力出版社, 2018.）

---

**发出任务书日期：2026 年 11 月 28 日　　　　指导教师签名：**

**预计完成日期：2027 年 5 月 27 日　　　　专业负责人签章：**

\**　　　　　　　　　　　　　　　　　　主管教学院长签章：\**

# 软件开发项目全生命周期管理 Agent — 实现思路（GitHub + 飞书 + Go 版）

> 本文在任务书基础上，将技术路线确定为：**代码托管平台 GitHub、即时通讯飞书、语言 Go、LLM 采用 API 调用模式（OpenAI 兼容协议）、Agent 核心自研（ReAct + Function Calling）、关系库 MySQL 8.0、向量库 Milvus**。
>
> 说明：任务书建议 PostgreSQL 与 Chroma，本方案改为 MySQL + Milvus，属于同定位替换（MySQL 8 的 JSON 类型同样满足"JSON 字段存 Git 事件原始数据"；Milvus 为工业级向量库，能力覆盖 Chroma）。替换理由与对比写入论文「技术选型对比」章节。

---

## 1. 确定后的技术选型总表

| 领域 | 选型 | 说明 |
|------|------|------|
| 语言 | Go 1.23 | 单二进制部署、并发模型适合事件处理 |
| LLM | OpenAI 兼容 API（DeepSeek / Qwen / GLM 任选） | `sashabaranov/go-openai` 客户端 |
| Agent 核心 | 自研 ReAct 循环 + 工具注册表 | 对应论文 ReAct/Toolformer 路线 |
| Git 平台 | GitHub（**Webhook 主通道** + REST API v3 轮询对账） | `google/go-github`，鉴权用 GitHub App 或 PAT；开发期用 smee.io 转发免公网，见 3.5 |
| IM | 飞书开放平台 | `larksuite/oapi-sdk-go`（v3 官方 SDK） |
| IM 消息通道 | 自建应用 + **长连接事件订阅**（收）+ `im/v1/messages` REST（发） | 免公网 IP、免加解密；群机器人 Webhook 通道已移除，见 4.1 |
| 看板数据源 | 飞书多维表格（Bitable）可选 | 满足任务书"项目看板多源数据" |
| 关系数据库 | MySQL 8.0（utf8mb4 + JSON 类型） | 领域状态、事件原始数据（JSON 列） |
| 向量数据库 | Milvus standalone（docker-compose 部署） | `milvus-sdk-go`，变更摘要/Issue 聚类 |
| 定时任务 | robfig/cron v3 | 替代任务书中的 Celery Beat |
| HTTP 框架 | chi | Webhook 接收、管理 API |
| 数据访问 | **GORM**（MySQL 驱动 go-sql-driver/mysql） | 开发效率高、关联/钩子/事务好用；使用规范见 6.11 |
| 缓存 / 限流（可选） | Redis | **非必需**，M7 后按需引入：LLM 响应缓存、推送/调用限流；不做事实源，见 6.12 |
| 日志 / 配置 | slog / env | 标准库优先 |

---

## 2. 总体架构

```
┌────────────────────────────────────────────────────────────┐
│                     外部平台（数据源/出口）                  │
│  GitHub(Webhook/API)   飞书(长连接事件/群机器人/Bitable)    │
└──────┬──────────────────────┬──────────────────────────────┘
       │ 事件流入              │ 事件流入 / 消息推送
┌──────▼──────────────────────▼──────────────────────────────┐
│ 接入适配层  github/  feishu/  （协议细节全部封装在此）        │
└──────┬──────────────────────────────────────────────────────┘
       │ 统一内部事件（domain event）
┌──────▼──────────────────────────────────────────────────────┐
│ 事件总线（内存 channel 起步）→ 事件处理器 → 领域状态更新      │
└──────┬──────────────────────────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────────────┐
│ MySQL：git_events(JSON) / tasks / bugs / reviews / releases │
│        / test_cases / reminders                             │
│ Milvus：knowledge 集合（变更摘要/Issue 向量 + 标量过滤）      │
└──────┬──────────────────────────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────────────┐
│ Agent 核心：LLM 客户端 + 工具注册表 + ReAct 循环 + 记忆      │
│ 触发器：IM 对话 / cron 定时 / 事件规则命中                   │
└────────────────────────────────────────────────────────────┘
```

分层原则：**协议归接入层，状态归存储层，决策归 Agent，业务归「触发器 + Prompt + 工具组合」**。

---

## 3. GitHub 侧详细设计

### 3.1 鉴权方式选择

| 方式 | 优点 | 缺点 | 建议 |
|------|------|------|------|
| PAT（个人令牌） | 最简单，一行配置 | 权限粒度粗、过期需手动换 | 原型期用 |
| GitHub App | 按仓库安装、5000 req/h（装到 org 更高）、可多租户 | 需要 App 私钥 + JWT 换 installation token，逻辑多一点 | **正式版用** |

建议：先用 PAT 打通，再升级 GitHub App（论文里可作为鉴权设计演进写一节）。

### 3.2 需要从 GitHub 采集哪些数据（按业务用途分组）

不论 Webhook 还是轮询，业务侧需要的事实就这些——**先明确"要什么"，再决定"怎么拿"**：

| 数据 | 支撑的业务能力 | REST v3 端点 |
|------|--------------|-------------|
| **PR 列表**（open，含 draft、创建/更新时间、合并状态） | CR 跟踪、日报 | `GET /repos/{o}/{r}/pulls?state=open&sort=updated` |
| **PR 审查人** | 「谁被请求审查、等待多久」 | `GET /repos/{o}/{r}/pulls/{n}/requested_reviewers` |
| **PR 审查记录** | 判断"审过了没、谁审的、是否 Approved" | `GET /repos/{o}/{r}/pulls/{n}/reviews` |
| **Issues**（open，含 labels、assignee） | Bug 跟踪、严重级判定 | `GET /repos/{o}/{r}/issues?state=open&since=` |
| **Issue 评论** | 活跃度、最新进展（判断"有无响应"） | `GET /repos/{o}/{r}/issues/{n}/comments?since=` |
| **Commits** | 变更摘要 → embedding → RAG | `GET /repos/{o}/{r}/commits?since=` |
| **Releases** | 版本发布跟进 | `GET /repos/{o}/{r}/releases` |
| **CI 运行状态**（可选加分） | 构建失败提醒 | `GET /repos/{o}/{r}/actions/runs` |
| **协作者/成员** | 账号映射（GitHub login ↔ 飞书） | `GET /repos/{o}/{r}/collaborators` |

**一个必踩的坑**：Issues 接口的返回里**混着 PR**（GitHub 把 PR 也视为 issue），每条数据带 `pull_request` 字段——必须在代码里过滤掉，否则 Bug 列表里全是 PR。

### 3.3 通道定位：Webhook 主通道 + 轮询对账（本项目采用）

**本项目采用 Webhook 作为主采集通道**，轮询降级为**对账与补漏通道**（详见 3.5 的实现与调试细节）。两者不是二选一，而是分工：

| 通道 | 角色 | 频率 | 解决什么 |
|------|------|------|---------|
| **Webhook**（3.5） | 实时主通道 | 事件到达即处理（秒级） | 实时性：PR 被请求审查、Issue 被指派、Release 发布 |
| **轮询**（本节） | 对账 / 补漏 / 建基线 | 15~30 分钟 | 稳健性：Webhook 漏投、服务停机期间的事件、瞬时状态的最终校正 |

之所以保留轮询而非"纯 Webhook"：Webhook 是**至少一次、不保证顺序、失败重投有限次**（GitHub 重投失败后就不再重试），且服务重启/穿透断线期间的事件会彻底丢失。轮询对账是唯一能把系统拉回一致的兜底手段——与 5.3 的 Outbox 对账是同一思想。

**两条通道产出同一种内部事件**（见 3.4 统一事件模型），下游处理器不知道数据从哪来，因此可以随时切换或双开，零改动。

#### 3.3.1 轮询的技术要点：拉取 → 快照 → 差分

**核心难点：轮询拿到的是"状态"，不是"事件"**。Webhook 直接告诉你「张三被请求审查 PR#12」；轮询只能看到「PR#12 的审查人列表 = [张三]」。要还原出事件，必须**与上一次快照做差分**：

1. 拉取当前快照 → 2. 与库中 snapshot 对比 → 3. 推断领域事件（审查人新增 = `ReviewRequested`；审查人消失且出现 review = `Reviewed`；Issue 由 open 变 closed = `IssueClosed`）→ 4. 产出与 Webhook 格式一致的内部事件投给总线。

**工程要点**：

- **首次全量、之后增量**：首次建基线快照，后续用 `since` / `sort=updated` 拉增量，配合 `sync_cursor` 表记每个仓库每类资源的 `last_synced_at`；
- **snapshot 表**：保存上一次的逐条状态（PR 的审查人集合、状态、更新时间），是差分的唯一依据，不可省略；
- **分页**：`per_page=100` 并处理响应头的 `Link`（next 页）；
- **配额省着用**：关注 `X-RateLimit-Remaining`，低水位时降频；用 `If-None-Match` 带 ETag 做条件请求，**返回 304 时不消耗配额**；
- **分级频率**：PR/Issue 15~30 分钟；Commits 30~60 分钟；Releases 数小时；CI 状态按需；
- **可回放**：轮询方案独有优势——随时可丢弃状态重拉全量重建，毕设演示与调 bug 时极其方便。

**按业务维度的采集观测点**（同一份快照，不同域关心不同字段）：

- **CR 维度**：重点观测 PR 的 `requested_reviewers` 集合变化与 `reviews` 新增，据此算"等待时长"；
- **Bug 维度**：观测 Issue 的 `labels`（P0/P1 判定）与 `assignee` 变更，以及 `comments` 是否有新增（判断"有无响应"）；
- **RAG 维度**：观测 `commits` 条数增长，新增 commit → 生成变更摘要 → embedding → Milvus（经 outbox，见 5.3）。

**容错与自动化策略**：

1. **配额自适应**：监控 `X-RateLimit-Remaining`，低于阈值（如剩余 < 50）自动延长下一轮轮询间隔；被限流封禁时打日志并退避，不重试猛冲；
2. **采集失败**：网络类失败采用指数退避重试（1min / 5min / 30min），超过上限记 `failed` 并告警；
3. **向量生成失败**：不阻塞主流程——落 `vector_outbox` 表，由独立 worker 按 `source_id` 幂等重试直到成功，保证 Milvus 与 MySQL 最终一致；
4. **幂等保证**：差分逻辑本身即幂等——只有字段真正变化才产出事件，重复拉取同一状态不会重复触发；每轮仅更新发生变化的 snapshot 字段。

**代价（诚实记录，写进论文局限性）**：延迟最多一个轮询周期（对日报无影响，对 P0 级实时提醒有影响，但这部分由 Webhook 主通道覆盖）；看不到瞬时过程（如"先 request changes 再 approve"）；已删除的数据（删掉的 issue/comment）拉不到；空轮询也消耗 API 配额。

**与飞书侧的不对称**：飞书是**出方向长连接**（你的程序主动连出去，无需公网），GitHub 是**入方向 Webhook**（GitHub 主动 POST 你的地址，必须有公网可达端点）。这是本项目网络拓扑上最容易被忽略的不对称点，也是 3.5 要解决的核心问题。

### 3.4 统一事件模型

接入层收到 GitHub 事件后**不直接处理业务**，而是翻译成内部领域事件（如 `PRReviewRequested`、`IssueAssigned`），带上统一字段（仓库、外部 ID、时间、角色、负载 JSON）丢进事件总线。好处：换 GitLab 只需新增一个适配器；处理器与 GitHub 协议解耦。

---

### 3.5 Webhook 实现细节与开发期调试方案（本项目主通道）

#### 3.5.1 基本原理

**Webhook = 反向 API / HTTP 回调**。通常由你主动调别人的接口；Webhook 反过来——你注册一个 URL，事件发生时**由对方主动 POST 给你**。相比轮询（你反复问"有事吗"，绝大多数是空响应），Webhook 是事件驱动推送：实时、零轮询开销、零配额浪费。

**一个 Webhook 请求的解剖**：

```
POST https://your-domain/webhook/github
Content-Type: application/json
X-GitHub-Event: pull_request          ← 事件类型（决定用哪个解析器）
X-GitHub-Delivery: 72d3162e-...       ← 投递 ID（去重用）
X-Hub-Signature-256: sha256=5f8...    ← HMAC 签名（验真用）

{ "action": "review_requested", "pull_request": {...} }   ← payload
```

你的服务应**尽快返回 2xx**（GitHub 的超时是 **10 秒**），实际处理异步化。

#### 3.5.2 三个必备工程项（漏一个就出事）

1. **验签**：用共享密钥对**原始请求体**做 HMAC-SHA256，`hmac.Equal` 常量时间比较。**必须用未经反序列化的原始字节**——先 `json.Unmarshal` 再重新序列化会导致签名永远对不上（字段顺序变化），这是最常见的卡壳点；
2. **防重放**：校验时间戳（`X-GitHub-Delivery` 无法表达时效，需用请求到达时间或 payload 内的时间），拒绝过期请求（如 5 分钟以外的直接丢弃）；
3. **幂等去重**：Webhook 是 **至少一次投递（at-least-once）**，失败会重投，且**不保证顺序**——用 `X-GitHub-Delivery` 建唯一索引去重，判断"新旧"要用**事件自带的时间戳**，不能依赖到达顺序（晚到的可能是更早发生的事件）。

**处理链路**：验签 → 写 `git_events`（原始 payload 全量存 JSON，投递 ID 唯一）→ 立刻返回 200 → 投事件总线 → 后台 worker 翻译成领域事件并异步处理。**绝不**在 HTTP handler 里跑 LLM 或写 Milvus。

#### 3.5.3 开发期调试：四种方案（推荐 ①→② 的路径）

核心矛盾只有一句：**GitHub 要能访问到你的机器，而开发机通常在内网/没有公网 IP**。

| 方案 | 原理 | 是否需要公网 | 真实事件 | 推荐度 |
|------|------|------------|---------|--------|
| **① smee.io 转发**（首推） | GitHub POST 到 smee 公网 channel，你本地跑一个客户端**主动连 smee** 把事件拉回来 | ❌ 不需要 | ✅ 真实 | ⭐⭐⭐⭐⭐ |
| **② 内网穿透**（ngrok / Cloudflare Tunnel / frp） | 把本地 8080 映射成公网 URL，填进 GitHub | ❌ 不需要（工具给） | ✅ 真实 | ⭐⭐⭐⭐ |
| **③ 云主机部署** | 直接部署到有公网的服务器 | ✅ 需要一台云主机 | ✅ 真实 | ⭐⭐⭐ |
| **④ 本地伪造 payload**（curl / httptest） | 自己构造带签名的请求打本地端点 | ❌ | ❌ 假的 | ⭐⭐⭐⭐（单测必配） |

**① smee.io —— 最贴合本项目"不出公网"的诉求**

它是 GitHub 官方文档推荐的调试工具，机理和飞书长连接**完全同构**：

```
GitHub ──POST──▶ smee.io 公网 channel ──(你的客户端主动建立的长连接)──▶ localhost:8080
```

关键点：**你的机器是出方向连 smee**（和飞书长连接一样），所以本地不需要公网 IP、不需要开防火墙。做法：

1. 打开 https://smee.io/new 拿到一个 channel URL（如 `https://smee.io/abc123`）；
2. 把这个 URL 填到 GitHub 仓库 Webhook 配置的 **Payload URL**（不是填你本地地址！）；
3. 本地运行转发客户端把它转到你本机端口：`npx smee-client --url https://smee.io/abc123 --target http://localhost:8080/webhook/github`（也有 Go 实现和 Docker 镜像，可避免依赖 Node）；
4. 在仓库里随手做个动作（开个 issue、request 个 review），本地就能收到真实 payload。

额外福利：smee 的 Web 页面能**实时展示每次投递的 headers 和 body**，排查字段结构时比看日志直观得多。

**② 内网穿透**：`ngrok http 8080` 拿到 `https://xxx.ngrok-free.app`，填进 GitHub 即可，最直白。注意免费版地址**每次重启会变**（要重新去 GitHub 后台改配置，或用付费固定域名）；Cloudflare Tunnel（`cloudflared`）可配固定子域名，长期更省心。

**④ 本地伪造 + GitHub 的 Redeliver（强烈推荐的组合技）**

不需要真触发事件，也能拿到**真实的 payload**：

1. 仓库 → Settings → Webhooks → 点进你的 webhook → **Recent Deliveries** 面板；
2. 每一次投递都能看到**完整的 request headers（含签名）和 body**，还有 **Redeliver** 按钮可以原样重发；
3. 把某次真实投递的 body 存成 `testdata/pr_review_requested.json`，写一个 `curl` 或 Go 单测脚本，本地算出正确签名后打给自己的 handler。

这样你拥有了一份**覆盖各类事件的真实样本集**，既能本地反复调试（改代码→重放→看结果，秒级循环），又能作为**单元测试的 fixture**——这是 Webhook 开发效率最高的方式，且不依赖任何网络工具。

**调试期的两个便利开关**（务必只在开发环境开启，用配置区分）：

- `WEBHOOK_SKIP_SIGNATURE=true`：跳过验签，避免手工 curl 时反复算签名；
- 一个 `POST /debug/replay` 端点：直接投递本地 JSON 文件，绕过网络。

#### 3.5.4 开发路线建议

1. **阶段一（零依赖）**：本地起 `chi` 路由 + handler，用 `httptest` + 保存好的真实 fixture 跑通"解析 → 落库 → 投总线"，验签逻辑单独测；
2. **阶段二（接真数据）**：上 smee.io，用真实事件验证签名、payload 结构与自己的假设是否一致（此时常会发现"原来这个字段是嵌套在里面的"）；
3. **阶段三（稳定化）**：部署到云主机或用固定域名的穿透，作为演示环境的正式端点；同时**始终开着轮询对账**（3.3），这样隧道断了、服务重启了都不会漏数据。

#### 3.5.5 官方约束与常见坑

- **10 秒超时**：超时会被记为投递失败，GitHub 会重投——所以 handler 必须快速返回；
- **Content type 选 `application/json`**（不要选 `application/x-www-form-urlencoded`，后者 body 是 `payload=` 编码形式，验签和解析都更麻烦）；
- **SSL 与重定向**：GitHub 要求 HTTPS，且不接受 302 重定向（穿透工具给的 URL 必须直接可用）；
- **事件订阅要显式勾选**：默认只有 `push`，PR / Issue / Release 都要在 webhook 配置里单独选（或用 `Send me everything`）；
- **`ping` 事件**：配置 webhook 时 GitHub 会先发一个 `ping`，你的 handler 要能识别并返回 200，否则 GitHub 会认为配置失败；
- **Secret 建议配**：不配也能收，但配了才能验证消息来源真实性（论文里"安全性设计"要有这一条）。

## 4. 飞书侧详细设计

### 4.0 IM 域的定位与它解决的问题

**飞书开放平台的 IM 业务域**（`im/v1`）是开放能力里最大的一块，覆盖：消息（发送/回复/撤回/转发/表情回复/已读）、会话与群（建群、拉人、群信息）、消息形态（文本、富文本、图片、文件、交互卡片）、以及配套事件（`im.message.receive_v1` 等）。它解决的本质问题是：**让一个应用以"群成员/联系人"的身份，在飞书里进行对话级交互**。配套域：contact（通讯录，取人名与身份）、bitable（多维表格，可作看板）。

**在本项目中 IM 承担三个不可替代的作用**：

1. **触达（打断式提醒）**：项目事实发生在 GitHub，但人的注意力在 IM。看板/邮件需要人主动去看，IM 是唯一能"把信息推到人面前"的通道——提醒的价值就在于打断。
2. **双向交互**：对话指令（@机器人 提问）与卡片按钮（指派给我/忽略），让系统从"只会播报"升级为"可操作"，形成 GitHub 事实与人类决策之间的闭环。
3. **上下文归属**：`chat_id` 提供会话上下文、`open_id` 提供身份、引用回复提供话题归属，使交互有明确的"谁、在哪、关于什么"。

**边界纪律（重要）**：IM 只是**通道**，不是业务存储。业务真值永远在 MySQL（任务状态、提醒去重、账号映射）；聊天记录仅作审计与回溯。禁止把"群里说过什么"当作系统状态来源——IM 消息可能丢失、重推、被编辑，不具备真值资格。

### 4.1 通道收敛为两条（群机器人通道已移除）

**最终决定：移除「自定义群机器人 Webhook」通道**，飞书侧只保留两条：

| 通道 | 方向 | 用途 | 前置条件 |
|------|------|------|---------|
| 自建应用 + 长连接事件订阅 | 入 | 用户 @机器人 对话、卡片按钮回调 | 开放平台建应用、开长连接模式 |
| 应用身份 REST API `im/v1/messages` | 出 | 群消息、私聊、卡片推送（日报/提醒） | 机器人进群或拿到 open_id |
| Bitable API（可选） | 读写 | 任务看板数据源 | 应用授权多维表格权限 |

**移除理由**：自建应用发消息（`receive_id=chat_id`）**完全覆盖**群机器人 Webhook 的推送能力，且额外具备——可发私聊、可 @指定人、卡片按钮回调可接回、一套凭据与一处权限管理。保留两条推送通道只会带来两套配置、两套署名、两套失败语义，收益为零。

### 4.2 自定义机器人：仅作为原型期的一次性验证工具

不进入正式架构。它唯一的价值是**最快验证链路**：群里加机器人拿到 URL，POST 一条 JSON 就能看到消息出现在群里（可同时校验卡片渲染效果），全程 5 分钟、不需要建应用和审批。**用于确认"网络通、格式对"，验证完即弃用**。

（实现要点：安全设置选「签名校验」，timestamp + 密钥做 HMAC-SHA256、Base64 后放 `sign` 字段。）

### 4.3 自建应用（对话与交互，核心）

1. 在开放平台创建**企业自建应用**，开启「机器人」能力；
2. 事件订阅模式选 **长连接（WebSocket）**——这是选飞书的一大优势：**不需要公网 IP、不需要处理 encrypt_key 加解密**，SDK 维持长连接自动收事件、断线重连。本地开发直接跑，毕设演示极其方便；
3. 订阅事件：`im.message.receive_v1`（收到消息/被 @）；
4. 用户 @机器人 → SDK 回调给你一个消息结构（chat_id、sender、text、mentions）→ 起一次 Agent Run → 用 `im.v1.message.reply` 回复；
5. 权限申请：`im:message`（收发消息）、`im:message.group_at_msg`（群 @）、后续卡片回调用 `card:action`、Bitable 需 `bitable:app` 等；
6. token 管理：`tenant_access_token` 有效期 2h，SDK 自动刷新，但要注意并发下别同时刷新（SDK 已处理，自研时要加锁）。

### 4.4 消息卡片交互（加分项，建议做）

提醒卡片带按钮：如 Bug 提醒卡片带「指派给我」「忽略」按钮 → 用户点击 → 卡片回调（长连接同样能收到 `card.action.trigger`）→ 你的处理器更新任务状态并回写卡片（把卡片改成"已指派给张三"）。这让系统从"只会通知"升级成"可操作"，答辩演示效果拔群。

### 4.5 飞书接入实现要点（SDK 与消息收发机制澄清）

**工具链澄清**：`lark-cli`（第三方 CLI）是给人用的终端工具（基于官方 SDK），**不是**服务运行时的依赖。本项目使用官方 SDK `github.com/larksuite/oapi-sdk-go/v3`。

**收发机制（勿混淆两种 webhook）**：

| 能力 | 机制 | 是否 webhook |
|------|------|-------------|
| 收消息（用户@机器人） | 事件订阅：长连接（SDK `larkws`，WS 由本服务主动建立）或回调 URL（HTTP） | 否 |
| 发消息/回复 | REST API `POST /open-apis/im/v1/messages`（SDK `larkim`，tenant_access_token） | 否 |
| 群机器人单向推送 | 自定义机器人的 Webhook URL（只能发、不能收，群级别配置） | 是（**本项目不采用**，见 4.1/4.2） |

**选型决定**：统一走**自建应用 + 长连接**，收与发都用应用身份（App ID/Secret），群机器人通道移除。

**工具链边界（易混淆）**：`lark-cli` 是第三方**命令行工具**（基于官方 SDK，供人在终端里操作飞书），**不能作为服务运行时的依赖**——它不是库、无法接收事件流、其认证流程面向交互式使用。程序内获取消息/文档一律走官方 SDK `oapi-sdk-go/v3`（或直接裸调开放平台 REST API，二者等价，SDK 只是封装）。`lark-cli` 的定位是**开发调试工具**：列群、发测试消息、查权限、验证 token，非常好用，但不进生产代码。

**实现骨架**：`larkws.NewClient(appID, appSecret, WithEventHandler(dispatcher))` → dispatcher 注册 `OnP2MessageReceiveV1` → `Start(ctx)` 阻塞运行；发送/回复使用 `larkim` 的 `Reply`（带 message_id，会话内回复，用于对话）与 `Create`（指定 chat_id/open_id 主动发，用于日报与提醒）。

**五个实战要点**：

1. **回调内不做重活**：事件到达先落库并返回，Agent 在后台 goroutine 执行，完成后用 API 主动回复，避免长连接事件超时重推；
2. **消息解析**：`message.content` 为 JSON 字符串（如 `{"text":"..."}`），群消息中 @机器人 会含 `<at user_id="ou_xxx">名字</at>` 标签，必须剥离后再交给 LLM；
3. **幂等去重**：长连接重连后可能重推，用 `message_id` 去重；
4. **只响应 @ 自己的消息**：校验 `mentions` 中是否含本机器人 open_id，避免群内噪音；
5. **长连接为集群模式、不广播**：同一应用多实例时仅随机一个实例收到事件（官方说明），单实例部署可行，但不能靠多实例分担事件负载——写入论文局限性章节。

SDK 细节：v3.9.7 起提供 `Client.Close()` 真正断开并关闭自动重连（早期版本 `Start` 内部 `select{}` 不响应 ctx 取消），优雅退出需使用该方法。

### 4.6 长连接建连机理与开发环境（为什么不需要公网）

**建连方向**：长连接是**出方向**连接——本机 SDK 主动向飞书开放平台建立 WebSocket 全双工通道，事件沿同一条连接回传。因此只需运行环境**能访问公网（出网）**，**不需要公网 IP、域名、内网穿透**，也不需要配置加密策略（SDK 内置加密与鉴权，无验签/解密代码）。对比 HTTP 回调模式是**入方向**（飞书 POST 你的地址），必须有公网可达地址且需处理验签解密。

**SDK 内部流程**（只需 `larkws.NewClient(appID, secret)` + `Start(ctx)`，以下由 SDK 完成）：获取 tenant_access_token → 获取 wss 端点 → 建立 WebSocket → 心跳维持 → 断线自动重连。连接成功日志出现 `connected to wss://xxx`，`Start` 阻塞主线程。

**官方约束（必须遵守）**：

1. **3 秒超时重推**：收到事件后需 **3 秒内处理完成且不抛异常**，否则触发超时重推——这从官方层面规定了"回调内只做解析与落库、Agent 异步执行"的设计（见 4.7 第 1 条）；
2. **仅支持企业自建应用**，商店应用不支持长连接；
3. **每应用最多 50 条连接**（每初始化一个 client 即一条连接）；
4. **集群模式不广播**：多客户端时随机一个收到事件，不能靠多实例分担事件负载；
5. **配置顺序坑**：在开放平台"事件与回调 > 事件配置"里把订阅方式改为"使用长连接接收事件"并保存时，**必须先启动本地客户端、保证有长连接在线**，否则保存不成功。

### 4.7 收发消息的实现链路

**收消息（长连接 → 事件总线 → Agent → 回复）六步**：

1. dispatcher 回调触发（运行在 SDK 的 goroutine 中）；
2. 解析：从 `P2MessageReceiveV1` 取 `message_id`、`chat_id`、`sender.sender_id.open_id`、`content`（JSON 字符串 → 反序列化取 text），剥离 `<at user_id=...>名字</at>` 标签，判断是否 @本机器人；
3. 落库去重：写入 `im_messages` 表，`message_id` 唯一索引，冲突即丢弃（幂等，防长连接重推）；
4. 立即返回回调（处理逻辑不在此 goroutine 内执行）；
5. 投递内部事件总线，后台 worker 起一次 Agent Run；
6. 结果经 `larkim` 的 `Reply`（带原 message_id）发送回复；失败按退避重试，超限落 outbox 由对账补发。

**发消息（主动推送）三类调用**：

- `MsgType=text` + content `{"text":"..."}`：纯文本，群内 @某人在文本里嵌 `<at user_id="ou_xxx">名字</at>`；
- `MsgType=interactive` + content 为卡片 JSON：日报、提醒、任务确认（带按钮）；
- 目标选择：`ReceiveIdType` 用 `chat_id`（发群）或 `open_id`（发私聊），**统一用 open_id 体系**，`user_id` 已逐步废弃。

**适配器内部结构**（`internal/ingest/feishuadapter/`）：

- `longconn.go`：建 `larkws.Client`、注册 `OnP2MessageReceiveV1`、事件翻译为内部 `IncomingMessage` 投给事件总线；`Start` 阻塞，需在 main 中另起 goroutine，退出时用 `Client.Close()`（v3.9.7+）；
- `sender.go`：封装 `SendText / SendCard / Reply / UpdateCard`，内部统一处理限流重试与失败落 outbox；
- `card.go`：卡片 JSON 构造器（卡片交互设计见 4.4 节）。

**必踩的坑**：

1. 权限：需在开放平台申请消息读写相关权限（`im:message` 系）与用户信息读取权限，具体权限点以控制台为准；仅申请不生效还需发布版本并管理员审批；
2. token：tenant_access_token 由 SDK 内部缓存与刷新，不要自己缓存或每次请求都换新；
3. ID 体系：chat_id / open_id / union_id 三者不同，跨应用统一身份用 union_id；
4. 回调耗时：处理逻辑必须异步化，回调内只做解析与落库；
5. 机器人 open_id：启动时调 bot info 接口获取自身 open_id 缓存，用于判断"是否被 @"。

### 4.8 飞书用户 ↔ GitHub 账号映射

提醒要推给"人"，但 GitHub 事件里只有 GitHub login。需要一张 **account_mapping** 表（飞书 user_id ↔ GitHub login ↔ 显示名），来源两种：管理员在配置里手填，或做「绑定」指令让用户发 `绑定 github mylogin` 由 Agent 写表。这是多平台系统里必踩的坑，提前设计掉。

---

## 5. 存储层详细设计（表结构字段级）

MySQL 约定：库与表的字符集一律 `utf8mb4`（默认 utf8 存不了部分中文/emoji，卡片文案必踩）；JSON 字段统一 MySQL 8 `JSON` 类型（读取用 `JSON_EXTRACT` / sqlx 扫到 `[]byte` 再反序列化）；所有表带 `created_at / updated_at`；外部对象存 `external_id` 做幂等；迁移用 `golang-migrate` 管理 SQL 文件。

### 5.1 事件与状态（MySQL）

- **git_events**：`id, event_type, repo, external_id, actor_login, payload(JSON), occurred_at, processed(bool), processed_at`。幂等键 `(event_type, external_id)` 唯一索引——Webhook 会重发，重复插入直接冲突忽略。
- **projects**：`id, name, repo_full_name, feishu_chat_id(日报推送目标), webhook_url, settings(JSON 如提醒阈值)`
- **pull_requests / reviews**（CR 跟踪模型）：`pr_id, repo, number, title, author, state, review_requested_at, reviewers(JSON), last_review_at, merged_at`。**提醒判定核心字段是 `review_requested_at`**——超时天数由它算出。
- **issues / bugs**：`issue_id, repo, number, title, labels(JSON), assignee, state, severity(规则从 label 推), opened_at, closed_at, last_comment_at, reminder_count`
- **releases**：`release_id, repo, tag, published_at, notes_url, checklist_status(JSON)`
- **tasks**（任务派发模型）：`id, project_id, title, description, parent_id(需求拆解出的父子关系), assignee_github, assignee_feishu, status(状态机: todo/doing/review/done), due_date, source(issue/PR/对话/手动)`
- **test_cases**：`id, task_id, title, steps(JSON), expected, priority, status(draft/approved/run/pass/fail), generated_by`

### 5.2 向量知识库（Milvus）

Milvus 是独立于 MySQL 的第二套存储，检索走两段式：**Milvus 算相似 → 拿 source_id 回 MySQL 查详情**。

- 集合 **knowledge**，字段：
  - `id`（INT64 主键，自增或雪花）
  - `source_type`（VARCHAR，标量过滤用：issue / pr / summary）
  - `source_id`（VARCHAR，回 MySQL 关联的键）
  - `text`（VARCHAR，原文摘要，检索结果可直接展示）
  - `embedding`（FLOAT_VECTOR，维度对齐所选 embedding 模型，如 1024）
  - `meta`（JSON，仓库名、时间等辅助过滤信息）
- 索引：`embedding` 建 HNSW（M=16, efConstruction=200），metric 用 `COSINE`；
- 检索：`source_type` 作为标量过滤条件传入 search（Milvus 原生支持 `source_type == "issue"` 的 bool expression），top_k 取 5；
- 一致性注意：Milvus 插入默认异步可见，插入后立刻搜索可能查不到——`search` 时指定一致性级别 `Strong`，或在入库流程中等待 flush 完成；
- 部署：docker-compose 拉起 Milvus standalone + etcd + MinIO 三容器（预留 4GB 内存），SDK 用官方 `milvus-sdk-go`。

### 5.3 双库一致性与向量入库（Transactional Outbox 模式）

MySQL 与 Milvus 之间没有事务、也无法用锁补出跨系统原子性，采用最终一致 + Outbox 兜底：

- **vector_outbox** 表（MySQL）：`id, source_type, source_id, text(待嵌入文本), status(pending/processing/done/failed), retry_count, next_retry_at, created_at`；
- 写入流程：业务事务内**同时**写业务行 + outbox 记录（同一事务，原子）→ 后台 worker 轮询 pending 记录 → 调 embedding API → 写 Milvus → 标记 done；
- 失败处理：状态机驱动重试（指数退避，如 1min/5min/30min，超过 5 次转 failed 人工介入）；
- **对账任务**（每小时）：统计 MySQL 中"应有向量"的业务行数 vs Milvus 实际条数，缺失或悬空的记录重新入队（source_id 反查）；
- 轮询实现的并发要点：多 worker/多协程取任务用 `SELECT ... FOR UPDATE SKIP LOCKED`（MySQL 8+），被锁行自动跳过，天然不重复消费；
- 规模化演进路径：当事件量增长，轮询 worker 可平滑替换为 Debezium 监听 binlog → 消息队列 → 消费者写 Milvus（CDC 形态），outbox 表结构不变；
- 设计边界说明：本系统需要强一致的数据（任务状态机、提醒去重、账号映射）全部为纯关系数据，MySQL 原生事务覆盖；跨库的只有 RAG 派生索引，允许分钟级延迟——**一致性要求与存储分界线天然对齐**，这是双库架构成立的前提。
- 异步是常态而非妥协：元数据是业务事实、向量是派生数据，派生永远滞后于源头。且 embedding API 调用（百毫秒~秒级）绝不应放进插入路径或数据库事务（长事务反模式）——即便用 pgvector，生产实践同样采用"先插行、向量后到"。
- **幂等**：Milvus 写入以 `source_id` 为业务主键做 upsert，outbox 重复消费不产生重复向量；
- **删除/更新传播**：业务行删除时置软删标记，对账任务负责清理 Milvus 孤儿向量；文本变更时重新 embedding 后按 `source_id` upsert 覆盖旧向量。这是双库方案相对单库（pgvector 同事务删除联动）需要额外付出的工作量。

### 5.4 提醒治理（防轰炸，工程上最容易被忽视的表）

- **reminders**：`id, target_feishu_user, subject_type, subject_id, reason(超时未审/Bug 超期/...), sent_at, channel`
- 唯一约束设计：同一 `(subject_id, reason, 日期)` 只允许一条 → 天然实现"每事每天最多提醒一次"；
- 再加一张 **reminder_policies**（每项目可配：哪些类型提醒、阈值小时数、免打扰时段），满足"基于规则与活动情况自动识别"的任务书要求。

---

## 6. 数据处理流水线设计（系统的核心）

前五章解决的是「数据从哪来、存到哪」。本章解决最关键的问题：**数据进来之后怎么变成有价值的动作**。这是整个系统真正的主体部分。

### 6.0 核心心智模型：一个逐级收敛的漏斗

先把量级算清楚，很多设计决策就自然浮现了（以一个中等活跃仓库的**每天**为口径估算）：

| 层级 | 数据规模 | 处理手段 | 单位成本 |
|------|---------|---------|---------|
| ① 原始事件 | 几百 ~ 几千条 JSON | 原样落库 | 磁盘，几乎免费 |
| ② 领域状态 | 几百行（PR / Issue / Task） | SQL 投影 | 极低 |
| ③ 规则候选集 | 几十条 | 确定性规则 / SQL | 极低 |
| ④ LLM 处理 | 几条 ~ 十几条 | 语言模型 | **高**（钱 + 秒级延迟） |
| ⑤ 最终动作 | 几条提醒 / 一条日报 | 去重后发送 | 低 |

**由此得出本章第一原则**：

> **越贵的处理越靠后，越靠后数据量越小。** 把 LLM 放在漏斗**末端**而不是入口，是整套设计里最能省钱、最保稳定、延迟最低的一条。

反面教材（很多初学者会这么写）：每收到一个 GitHub 事件就丢给 LLM"分析一下这个事件要不要提醒"——每天几千次 LLM 调用，其中 99% 的结论是"不用提醒"。正确做法是先用一条 SQL 把候选集筛到几十条，只对这些调 LLM。

**漏斗思想的第二个推论**：每一层的输出必须是**下一层的精确输入，且规模更小**。所以每一层都要能回答"我这一层筛掉了什么、为什么筛掉"——这既是调试依据，也是论文里"处理效率"章节的数据来源（比如"规则层过滤掉了 96% 的事件，LLM 调用量下降两个数量级"）。

### 6.1 五段主流水线 + 一条旁路

```
原始事件 (raw)
    │
    ▼
① 摄入 Ingest ──────── 验签 / 鉴权 → 原样落 raw 表 → 立即 ACK
    │
    ▼
② 规范化 Normalize ─── 协议翻译 → 内部领域事件 → 时区/ID/账号统一 → 幂等
    │
    ▼
③ 状态投影 Project ─── 事件 → 领域表（PR/Issue/Task 状态机），含乱序校正
    │
    ▼
④ 规则筛选 Filter ──── 纯确定性计算：超时？严重级？是否该提醒？→ 候选集
    │
    ▼
⑤ LLM 处理 Reason ──── 只对候选集做：归纳 / 生成 / 润色 / 检索增强
    │
    ▼
⑥ 动作输出 Act ─────── 发消息 / 写回 GitHub / 改任务状态（一律 outbox + 幂等）

   〔旁路〕派生数据管道：commit / issue → 摘要 → embedding → Milvus（异步，不影响主线）
```

配套的调度形态有两种，要区分开：

- **事件驱动**：①~③ 由 Webhook 或轮询差分触发，实时；
- **周期驱动**：④⑤⑥ 中的「批量型任务」（日报、超时扫描）由 cron 触发，是**先查库、再处理**，与事件无关。

### 6.2 第一段：摄入——只做三件事，多一件都不做

1. **验签/鉴权**：`X-Hub-Signature-256`（见 3.5）；
2. **原样落 raw 表**：把完整 payload 写进 `git_events`，**不做任何解析、不做任何业务判断**；
3. **立即返回 2xx**：GitHub 只等 10 秒。

**为什么 raw 表要保留完整 payload 且不可变**：它是你系统的**事实档案（system of record of facts）**。后面任何一层逻辑写错了（比如状态机算错、提醒漏发），你都可以从 raw 表**重放**重建一切，而不用去求 GitHub 再发一遍（Webhook 事件过期就没了）。这一个决策能让整个系统的可调试性提升一个量级，代价只是几十 MB 磁盘。

### 6.3 第二段：规范化——屏蔽外部协议的差异

把 GitHub 协议翻译成内部领域事件（3.4 统一事件模型），同时在这一层一次性解决三个"脏活"：

- **时区统一**：GitHub 返回 UTC，飞书要北京时间，在接入层统一转 `Asia/Shanghai` 后落库（避免后面到处 `+8`）；
- **ID 统一**：外部 ID 一律存 `external_id`，内部用自增主键，两者分开（外部 ID 可能变，且换平台就不同）；
- **账号映射**：GitHub login → 飞书 open_id（见 4.8），在这一层补全，让后续所有环节都能直接拿到"要提醒谁"。

**幂等在第二段收口**：`git_events` 上 `(event_type, external_id)` 唯一索引 + 投递 ID 去重，重复事件在第二段就被丢弃，不往下游走。

### 6.4 第三段：状态投影——事件变成可查询的状态

这一步是**事件溯源（Event Sourcing）思想**的轻量应用：领域表（PR / Issue / Task）不是直接写入的，而是**由事件投影出来的**。

```
事件流：ReviewRequested(#12, 张三) → ReviewSubmitted(#12, 张三, CHANGES_REQUESTED) → ReviewRequested(#12, 张三) → ...
                                          ↓ 投影
状态：  pull_requests #12: 状态=待修改, 当前审查人=[张三], 最后审查时间=..., 等待时长=3天
```

**三个必须处理好的细节**：

1. **乱序校正**（最容易出 bug 的地方）：Webhook 不保证顺序，晚到的可能是更早发生的事件。处理办法是**用事件自带的时间戳与库中记录的 `updated_at` 比较**，旧事件直接丢弃而不覆盖新状态。绝不能"谁后到听谁的"。
2. **状态机显式化**：PR（open → reviewing → changes_requested → approved → merged/closed）、Task（todo → doing → review → done）。状态转移在代码里写成显式的表（非法转移直接拒绝并告警），而不是散落各处 `if` 改字段——这个表本身就是论文里可以画的状态图。
3. **事务边界**：以**一个聚合**（如一个 PR）为单位做投影，一个聚合的若干事件在一个 MySQL 事务内处理完。这样即使中途崩溃，也不会出现"PR 状态改了但审查人没更新"的半成品。

### 6.5 第四段：规则筛选——能不花钱就不花钱

这一层是**纯确定性计算**，一行 SQL 就能出候选集，零 LLM 成本、零延迟、结果可预期：

```sql
-- CR 超时候选：已请求审查超过 48h、仍未有审查记录、且今天还没提醒过
SELECT ... FROM pull_requests p
LEFT JOIN reminders r ON r.target_type='pr' AND r.target_id=p.id AND r.sent_date=CURDATE()
WHERE p.state='open' AND p.review_requested_at < NOW() - INTERVAL 48 HOUR
  AND p.last_review_at IS NULL AND r.id IS NULL;
```

这一层承担的判断包括：是否超时、严重级（从 label 推）、是否已提醒过（去重表）、是否在免打扰时段、是否有对应的飞书账号（没人认领就不发）。

**设计纪律（写进论文很好看的一条）**：

> 凡是能用 SQL 或 if 判断得出结论的，一律不交给 LLM。LLM 只负责**需要理解自然语言**的部分。

这条纪律带来三个实际收益：省钱（调用量降两个数量级）、稳定（规则结果可复现，LLM 有随机性）、快（毫秒级 vs 秒级）。

### 6.6 第五段：LLM 处理——只做四件事

到了这一层，候选集已经很小了。LLM 只承担四类任务：

| 类型 | 例子 | 输入 | 输出 |
|------|------|------|------|
| **归纳** | 日报 | 几十条结构化状态 | 一段人话总结 |
| **生成** | 需求拆解、测试用例 | 一段需求描述 | 结构化任务/用例列表（JSON） |
| **润色** | 提醒文案 | 一条候选 + 上下文 | 自然、得体的提醒语 |
| **检索增强** | Bug 提醒附历史相似问题 | 候选 + 向量召回结果 | 带上下文的判断 |

**批处理 vs 逐条处理**的选择：

- **日报 = 批处理**：把一天所有状态**一次**喂给 LLM，让它归纳成一段。不要逐条调 LLM 再拼接（又贵又失去全局视角）；
- **提醒 = 逐条润色**：但要注意——如果文案高度模板化，**干脆用模板**，LLM 只在"需要个性化"时介入（比如给同一人的多条提醒合并成一段）。很多场景下模板 + 变量就够了，这块可以做成可配置开关，正好又是一组对比实验。

### 6.7 第六段：动作与输出

动作分三类，可靠性要求不同：

| 动作 | 例子 | 可靠性做法 |
|------|------|-----------|
| 内部写 | 改任务状态、记提醒 | MySQL 事务，直接做 |
| 外部写（IM） | 发飞书消息 | outbox + 退避重试 + 幂等（message_id/业务键） |
| 外部写（GitHub） | 回写评论、打标签 | 同上，且**必须显式开关**（默认只读更安全） |

前面积累的思想在这里再次复用：**任何跨越进程边界的写操作，都走「持久化意图 + 幂等重试 + 对账」**。

### 6.8 旁路：派生数据管道（向量化）

与主流程**完全解耦**，不阻塞任何主线环节：

```
commit / issue 落库 → 写 vector_outbox(pending) → worker 异步：
   生成摘要(LLM) → embedding(API) → upsert Milvus → outbox 置 done
                                    ↘ 失败：退避重试；仍失败：每小时对账补
```

关键是它在**主链路之外**（详见 5.3）：即使 Milvus 挂了一整天，提醒、日报、状态跟踪全部照常工作。这就是把派生数据剥离出去的价值。

### 6.9 贯穿全链路的三条性质

1. **幂等**：每一层都能重复执行而不产生副作用（唯一索引去重、状态机拒绝重复转移、外部写按业务键 upsert）。有了幂等，才敢重试；
2. **可观测**：`git_events.processed` 标记处理进度、`agent_runs` 记录每次 LLM 调用的 token/耗时/工具轨迹、`reminders` 记录发过什么。出问题能立刻定位到层；
3. **可重放**：raw 表保留 → 状态可重建 → 处理逻辑改了能重跑历史数据。**这三条是"能放心改代码"的前提**——没有它们，你每改一行逻辑都怕弄脏数据。

### 6.10 处理单元的划分：按聚合，不按事件

这是一个很关键但容易忽略的设计决定：**处理的单位不是"一条事件"，而是"一个聚合"**。

- 聚合例子：`(repo, pr_number)`、`(repo, issue_number)`、`task_id`；
- 为什么要按聚合：判断"PR#12 该不该提醒"需要它的**完整状态**，而完整状态由多条事件共同决定。逐条事件独立处理，无法做整体判断；
- 并发模型：**同一聚合串行、不同聚合并行**。实现上用一致性哈希把聚合 key 分到固定的 worker（或用一个按 key 分片的队列），这样既避免竞态，又能并行。这是 Go 里很自然的一个模式（map[shard]chan + 每 shard 一个 goroutine）。

### 6.11 GORM 使用规范（本章的实现约定）

确定使用 GORM，为避坑约定以下几条（都是实战里真会踩的）：

1. **事务一律用 `db.Transaction(func(tx *gorm.DB) error {...})`**，内部只用 `tx`；返回 error 自动回滚。不要手动 Begin/Commit 后忘记处理 error 路径；
2. **禁止裸 `Find(&x)` 全字段查询**大表：`git_events` 的 payload 可能几 KB，列表查询必须 `Select("id, event_type, occurred_at")` 指定列；
3. **警惕 N+1**：关联查询用 `Preload`，且不要在循环里查库（批量场景先一次 `Where("id IN ?", ids)` 取出再在内存里组装）；
4. **软删除与唯一索引冲突**：GORM 的 `gorm.DeletedAt` 软删后，唯一索引仍然生效——幂等去重这类**必须唯一**的表（如 `git_events`、`reminders`）**不要用软删除**，或把唯一键与 `deleted_at` 联合；
5. **迁移不要用 `AutoMigrate` 上生产**：它会改结构但管不了数据迁移，且不可回滚。正式用 `golang-migrate` 或 `atlas` 写版本化 SQL 迁移文件，AutoMigrate 只在本地开发图快；
6. **JSON 字段**：用 `datatypes.JSON`（gorm.io/datatypes）或自定义 `Scan/Value` 类型；查询 JSON 内部字段用 MySQL 的 `JSON_EXTRACT`；
7. **批量写**用 `CreateInBatches(batch, 100)`，别在循环里单条 Create；
8. **开慢查询日志**并设 `PrepareStmt: true`（默认已开，不要关），开发期打开 `logger.Info` 看生成的 SQL，能提前发现 N+1；
9. **连接池**：`sqlDB.SetMaxOpenConns / SetMaxIdleConns / SetConnMaxLifetime` 显式设置，否则默认值在高并发下会打满 MySQL 连接。

### 6.12 Redis：是否引入的诚实评估

**结论：本项目不是必需，建议放到 M7 之后按需引入。** 先说清楚判断依据——Redis 的四个真实价值点在你的系统里分别是：

| 场景 | Redis 的价值 | MySQL 能不能替代 | 建议 |
|------|------------|----------------|------|
| **LLM 响应缓存**（同问题短期复用） | 高，省钱明显 | 能（加缓存表），但查询开销大 | ✅ 值得引入 |
| **限流**（飞书推送频率、LLM 调用速率、GitHub API 配额） | 高，原子计数是它最擅长的 | 勉强（自增 + 行锁，有热点） | ✅ 值得引入 |
| **幂等前置拦截**（事件去重快速失败） | 中，减少 DB 压力 | **能**（唯一索引是最终保障） | ⚠️ 可选，且**不能替代**唯一索引 |
| **会话上下文 / 短期记忆** | 中 | 能（存 `agent_runs` 或会话表） | ⚠️ 可选 |
| **任务队列**（替代轮询 outbox） | 中 | 能（outbox 已够用） | ❌ 不建议，会破坏事务性优势 |

**几条必须守住的边界**（引入时别用错）：

- **Redis 永远不做事实源**。它是缓存和加速器，数据丢了系统必须仍能正确运行（Redis 挂掉时降级为直接查库，而不是报错）；
- **幂等的最终保障仍是 MySQL 唯一索引**。Redis 的 `SETNX` 只能做前置快速拦截，它无法提供事务性保证——这一点和前面讨论的"跨系统无原子"是同一个道理；
- **缓存要有失效策略和版本键**。LLM 的 prompt 模板改了，缓存键要跟着变（在 key 里带模板版本号），否则你会读到"旧 prompt 生成的结果"，排查起来非常隐蔽。

**引入时机建议**：先把 MySQL 版本全链路跑通（此时零 Redis 依赖，部署最简单），等 M7 提醒引擎上线、发现"同一问题反复调 LLM"或"推送频率需要限速"的真实痛点后，再加 Redis 解决具体问题。**为一个明确的问题引入组件，比"先加上以备不时之需"更专业，论文里也更好解释**。

---

## 7. Agent 核心详细设计

### 7.0 双实现路线：Eino 框架版 与 自研版（一个接口、两个实现）

**Eino 的 ReAct Agent 是什么**：官方实现在 `flow/agent/react`，`react.NewAgent(ctx, &react.AgentConfig{ToolCallingModel, ToolsConfig, MessageModifier, MaxStep, ToolReturnDirectly, ...})`。内核确为「ReAct 循环 + 工具注册」——底层 `compose.Graph` 只有 ChatModel 与 Tools 两个节点，历史消息存于 state，经 `MessageModifier` 处理后喂模型，循环至模型不再返回 tool call。模型侧通过 `ToolCallingChatModel.WithTools()` 绑定工具表（需支持 tool call 的模型实现，`eino-ext` 提供 openai / ark 等）。

Eino 在此内核之外额外提供：编译期类型检查、流式处理、并发安全的 state 读写、回调切面（模型调用前后注入埋点）、以及 ADK 多智能体模式（如 host 模式）的扩展空间。

**本项目采用双实现，通过一个接口切换**：

```
type Runner interface { Run(ctx context.Context, input string, ...) (Result, error) }
    ├── selfbuilt.Runner   // 自研 ReAct 循环（约 300~500 行）
    └── eino.Runner        // 包装 flow/agent/react.NewAgent
```

两条纪律：

1. **工具业务逻辑只写一份**：核心逻辑写成普通函数，自研侧包一层 JSON Schema 定义，Eino 侧包一层 `tool.InvokableTool` / `schema.ToolInfo`；
2. **配置切换**：`AGENT_ENGINE=selfbuilt|eino`，两个引擎共用同一批工具实现与业务代码。

**对照实验（论文实验章节素材）**：同一批真实指令下比较两引擎的——工具调用准确率、平均循环步数、端到端延迟、token 消耗、工具失败后的恢复行为。直接形成有自有数据的实验章节。

**推进顺序**：M2 先自研跑通（掌握原理）→ M4 后引入 Eino 实现第二个 Runner → 后期跑对照实验收集数据。

### 7.1 消息与循环结构

一次 Agent Run =：

```
输入（用户消息 / 定时任务注入的上下文摘要 / 事件触发注入的事实）
  → 组装 system prompt + 工具清单
  → LLM 请求（tools 数组）
  ← 返回 tool_calls（0~n 个）
  → 逐个执行工具（结果截断到安全长度）
  → 工具结果作为 role=tool 消息追加
  → 再次请求 LLM…… 循环
  ← 模型不再调工具、给出 content → 结束，输出交给通道（回复 IM / 生成卡片）
```

**护栏设计**（生产可用与玩具的分界线）：
- 最大循环步数（如 8 步），超限强制收敛："请基于已有信息直接作答"；
- 单工具执行超时与失败降级：把错误信息如实回给模型（它通常能自己换路），但连续失败 3 次终止；
- 工具结果大小限制：PR diff 动辄几万 token，工具内部先截断/摘要再返回；
- 全链路 trace：每次 Run 落一张 **agent_runs** 表（输入、每步的工具调用与结果、token 消耗、耗时）——既是调试利器，也是论文"测试验证"章节的原始数据。

### 7.2 工具注册表

每个工具 = `名称 + 描述 + 参数 JSON Schema + 执行函数 + 权限标记（只读/写）`。注册中心把所有工具 schema 序列化进每次 LLM 请求。**写类工具（create_task、send_notification）执行前可加确认策略**：对话场景先展示给用户确认，定时场景放行。

### 7.3 工具清单（按域分组）

**GitHub 只读**：`list_open_prs(repo, state过滤)`、`get_pr_detail(number)（含 reviewers 与等待时长）`、`list_open_issues(labels过滤)`、`get_issue_detail`、`get_recent_commits(limit)`、`list_releases`。

**飞书写**：`send_feishu_card(chat_id, card)`、`reply_message(message_id, text)`。

**领域读写**：`query_project_status(project)`（聚合任务/PR/Bug 概览，工具内部直接 SQL 聚合，别让模型自己拼数据）、`create_task(...)`、`assign_task(task_id, github_login)`、`update_task_status`、`save_test_cases(task_id, cases)`。

**RAG**：`search_similar(text, source_type, top_k)`（Milvus HNSW 余弦检索 + 标量过滤，拿到 source_id 后回 MySQL 补详情）。

**辅助**：`get_current_time`（调试用）、`lookup_user(query)`（名字→飞书/GitHub 账号，配合 4.5 的映射表）。

经验法则：**工具数量控制在 15 个以内**，模型选择准确率随工具数下降；给每个工具的描述里写清"什么时候该用我"。

### 7.4 记忆与上下文

- **短期**：本次 Run 的消息数组；上下文超预算时做"保留 system + 最近 N 轮 + 早期轮次摘要"的滑动窗口；
- **长期**：领域库即记忆。Agent "记得"某 Bug，本质是调 `get_issue_detail` 查库——不要幻想维护一个隐藏向量人格记忆，对毕设无必要；
- **会话续接**（进阶）：IM 群对话按 `chat_id` 存最近 20 条，注入为历史上下文。

---

## 8. 五大业务工作流详细走查

### 8.1 项目状态日报（cron，每天 09:00）

1. cron 触发 → 处理器直接跑 SQL 聚合出「事实清单」：今日新增/关闭 Bug、待审 PR 明细（含等待时长）、任务状态分布、近 24h commits；
2. **事实清单（不是原始数据）** 注入 prompt，要求 LLM 生成结构化日报（亮点、风险、建议关注）；
3. 调 `send_feishu_card` 推送到项目群卡片；
4. 日报落库存档（`daily_reports` 表），支持"给我看上周三的日报"这类回溯查询。
   要点：**聚合用 SQL、归纳用 LLM**——这是任务书参考文献[1]的 workflow 思想。

### 8.2 Code Review 超时提醒（cron，每小时 + 事件触发）

1. 事件 `review_requested` 到达 → 更新 pull_requests 表；
2. 每小时扫描：`review_requested_at` 距今 > 阈值（默认 24h）且该 reviewer 未提交 review 且今日未提醒过（查 reminders 唯一约束）→ 命中；
3. 对命中集合，一次 Agent Run：输入事实（PR 标题、作者、等待时长、diff 规模摘要），生成个性化提醒文案，调 `send_feishu_card` 发给 reviewer 的私聊/群 @；
4. 卡片带「去审查」按钮（链接 PR）与「今日免打扰」按钮。
   要点：**规则筛选在 LLM 之前**——每小时扫描 1000 个 PR，只有 3 个命中才调 LLM，成本和延迟都可控。

### 8.3 Bug 跟踪提醒

- Issue 打开且带 bug 类 label → 入 bugs 表，severity 由 label 映射（P0/P1…）；
- 扫描规则：P0 超 4h 未 assign、一般 Bug 超 48h 无评论 → 提醒 owner 或群；
- 用 `search_similar` 检索历史相似 Bug 附在提醒里（"3 个月前 #132 类似问题，修复 PR 见 …"）——这是 RAG 的杀手级应用点；
- Issue 关闭事件 → 更新状态，若存在关联未跑测试用例则提示。

### 8.4 需求拆解与任务派发（对话式）

用户在飞书 @机器人："把'支持导出周报为 Excel'这个需求拆一下"。
1. Agent Run：模型先调 `get_recent_commits`、`query_project_status` 了解现状 →（体现 Agent 的"先查再做"）；
2. 生成任务树（父需求→子任务，每条含描述/验收标准/建议负责人/预估）→ 调 `create_task` 落库（状态 draft）；
3. 回复卡片列出任务清单 + 每条带「指派」按钮；
4. 点击按钮或回复"同意指派" → `assign_task`，状态转 todo，可选再由机器人去 GitHub 建 Issue（`create_issue` 工具，GitHub App 权限）实现任务派发闭环。
   要点：**拆解结果必须先落库为 draft、人确认后生效**——LLM 产出不直接进入执行链路，这是可信 Agent 的基本盘。

### 8.5 版本发布跟进与测试用例

- `release published` 事件 → Agent Run：检查该 tag 涉及的任务完成度、关联 test_cases 的 pass 比例，输出发布检查单卡片（绿√/红×逐项）；
- 测试用例生成：对话或任务卡片触发 → 读任务描述与代码 diff 摘要 → 生成用例草稿（steps/expected）→ `save_test_cases(status=draft)` → 卡片确认后转 approved。执行结果由人回填或对接 CI（`check_run` 事件）更新 pass/fail。

---

## 9. RAG 流水线细节

- **入库时机**：push 事件 → 取 commits messages + PR 描述 + diff 统计，LLM 生成一段 100~200 字「变更摘要」→ embedding → 写入 Milvus knowledge 集合（source_type=summary，source_id 关联 MySQL 记录）；Issue 关闭时同理生成「问题与解决方案摘要」入库；
- **切分策略**：这些摘要天然短小，按条入库即可，不需要复杂 chunking——**先做简单的，够用再演进**；
- **检索时机**：Bug 提醒（相似问题）、CR 提醒（同文件历史变更）、对话问答（"这个模块之前为什么改过"）；
- **embedding 模型**：选一个国产 API 的 embedding 接口即可（通义/BGE API 等），维度对齐 Milvus collection 的 FLOAT_VECTOR 定义；入库与检索必须用同一个模型。

---

## 10. Go 项目目录结构（建议）

```
.
├── cmd/agent/main.go          # 装配：配置、DB、通道、调度器、启动
├── internal/
│   ├── config/                # env 加载
│   ├── llm/                   # LLM 客户端封装（重试/超时/计数）
│   ├── agent/                 # ReAct 循环、工具注册表、运行记录
│   │   └── tools/             # 每域一个文件：github.go / feishu.go / domain.go / rag.go
│   ├── ingest/                # 事件总线 + 处理器注册
│   │   ├── githubadapter/     # Webhook 验签、事件翻译、API 轮询
│   │   └── feishuadapter/     # 长连接、消息/卡片事件翻译、发送封装
│   ├── pipeline/              # 数据处理流水线（第 6 章）
│   │   ├── projector/         # 事件 → 领域状态投影（含状态机与乱序校正）
│   │   ├── filter/            # 规则筛选：超时/严重级/去重 → 候选集
│   │   └── shard/             # 按聚合 key 分片，同聚合串行、不同聚合并行
│   ├── domain/                # 实体与 GORM repository（约定见 6.11）
│   ├── cache/                 # Redis（可选）：LLM 响应缓存、限流器
│   ├── scheduler/             # cron 任务定义
│   ├── workflows/             # 日报/提醒/发布检查等流程编排（prompt 模板也放这）
│   └── card/                  # 飞书卡片 JSON 构造器
├── migrations/                # SQL 迁移文件（golang-migrate，不用 AutoMigrate）
└── doc/
```

依赖方向：`workflows → pipeline → agent → tools → domain/llm`；adapter 只进 `ingest`，永远不被业务直接引用。流水线三层（projector / filter / workflows）严格单向，不反向依赖 agent。

---

## 11. 开发里程碑（每步可独立验证）

| 阶段 | 内容 | 验收标准 |
|------|------|---------|
| M1 | LLM 客户端 + 命令行聊天 | 终端对话正常，含重试与 token 统计 |
| M2 | **Agent 循环 + 假工具** | 模型正确调用 `get_current_time` 并组织回答；步数护栏生效 |
| M3 | GitHub Webhook + 验签 + 事件落库 | smee.io 收真实事件，git_events 出现记录，重发同一 delivery 不重复；本地 fixture 回放单测通过 |
| M4 | 事件处理器 + 领域表 + PR/Bug 状态维护 | PR 开/审/合并全流程库内状态正确 |
| M5 | 飞书群机器人推送 + 日报 cron | 每天定时群里收到卡片日报 |
| M6 | 飞书自建应用长连接对话 | @机器人 能对话，Agent 调 GitHub 工具回答"现在有哪些待审 PR" |
| M7 | 提醒引擎（规则+去重+LLM 润色） | 构造超时 PR，收到提醒且当天不重发 |
| M8 | RAG（摘要入库+相似检索） | 新 Bug 提醒附带历史相似问题 |
| M9 | 需求拆解/任务派发/测试用例对话式闭环 | 拆解→确认→指派全流程走通 |
| M10 | 真实项目试用 + agent_runs 数据统计 → 测试报告 | 论文素材齐备 |

M2 是地基，务必最先做扎实；M5 之前不碰任何 LLM 业务功能。

---

## 12. 风险与坑清单

1. **Webhook 签名失败**：九成是用 JSON 反序列化后的 body 重算签名——必须保存原始字节；
1.5. **Webhook 通道天然会漏**：穿透断线、服务重启期间的事件 GitHub 重投有限次后就丢弃——**必须靠 3.3 的轮询对账兜底**，这也是本项目坚持双通道的原因；
2. **GitHub 限流**：PAT 5000/h，聚合工具别在循环里逐条调 API，一次列表 + 精选详情；
3. **飞书 token**：tenant_access_token 2h 过期，SDK 自动刷新即可，但自己缓存别每次请求都换新；
4. **长连接掉线**：SDK 有重连，但要加健康检查与进程级监控，挂了要能报警（可先打日志）；
5. **时区**：GitHub API 返回 UTC，飞书要北京时间，全部在接入层统一转 Asia/Shanghai 再落库；
6. **提醒风暴**：靠 reminders 唯一约束 + 免打扰时段兜底，上线第一天先灰度（只发给自己）;
7. **token 成本**：聚合走 SQL、LLM 只做归纳；每次 Run 的 token 落库，一周看一次账单曲线；
8. **模型工具误判**：工具描述写清边界（"仅当需要当前时间时调用"），并在评测集里覆盖"不该调工具"的用例；
9. **MySQL 字符集**：库/表/连接三处都要 utf8mb4（连接串加 charset 参数），否则中文或卡片符号插入报错；
10. **Milvus 可见性**：插入是异步的，插入后立刻 search 可能搜不到新数据——检索用 Strong 一致性或入库流程等 flush； Milvus standalone 重启后需要重新 load_collection；
11. **双库一致性**：MySQL 与 Milvus 之间无事务，采用 5.3 节的 Outbox 模式 + 每小时对账任务兜底，禁止业务代码直接双写（一律经 outbox）；

---

## 13. 与论文的对应关系

- 系统架构 → 第三章 系统设计（分层架构图、技术选型对比表）；
- MySQL + Milvus 替代任务书建议的 PostgreSQL + Chroma → 技术选型对比章节素材（JSON 能力、部署复杂度、向量检索能力、团队熟悉度四维对比表）；
- Agent 循环/工具/记忆 → 对应 ReAct[4]、Toolformer[5] 的实现落地；
- 规则筛选+LLM 归纳的分层 → 对应 Anthropic Building Effective Agents[1] 的 workflow/agent 辨析；
- agent_runs 表数据 → 第五章 测试验证（工具调用准确率、任务完成率、提醒时效性、用户反馈）；
- GitHub+飞书多源接入 → 佐证"多源数据自动汇总项目状态"的研究内容。

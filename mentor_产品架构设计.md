# Mentor 产品架构设计

> **用途**：基于 `mentor_MVP_PRD.md`（v0.9）落地为可评审、可开发的产品架构方案。
> **版本**：v0.1　**日期**：2026-08-30　**上游**：`mentor_MVP_PRD.md`、`mentor_导师对话_Prompt规格.md`
> **阅读对象**：研发（总体架构/服务拆分）、产品（功能架构对齐）、安全合规、测试
>
> **标注约定**：沿用前序文档——【事实】【推断】【决定】为产品/架构决策，【待研发确认】为技术选型开放项（不视为既定承诺）。

---

## 1. 架构设计原则（决定）

| # | 原则 | 落地含义 |
|---|---|---|
| P1 | **安全合规是一等公民，不是附加模块** | 未成年人模式、AI 标识、危机处置、数据保护作为横切能力内建于服务链路，而非事后补丁（PRD 第 3、9 章硬门槛） |
| P2 | **AI 能力可评估、可回滚** | 对话引擎带版本化 Prompt 与评估服务，任何 Prompt/模型变更先过评测集再上线 |
| P3 | **孩子体验优先，家长透明兜底** | 孩子端极简、对话为主入口；家长端价值（周报/风险）由异步服务保障，不阻塞孩子主流程 |
| P4 | **数据最小化与分级保护** | 儿童情绪、家庭等敏感信息单独加密、最小权限、可删除（PRD 3.4） |
| P5 | **双通道安全兜底** | 危机识别 = 模型识别 + 规则/关键词双通道，人工复核兜底（PRD 4.9） |
| P6 | **模块化可替换** | LLM/ASR/审核等外部依赖抽象为接口，避免供应商锁定（规划文档 4.2） |

> 【待研发确认】具体技术栈、中间件、云服务供应商由研发在评审后确定；本文给出职责边界与候选方向，不承诺具体产品。

---

## 2. 总体架构（分层视图）

```mermaid
flowchart TB
    subgraph L1_客户端["客户端层"]
        C1["孩子端 App<br/>iOS/Android"]
        C2["家长端 App<br/>iOS/Android"]
        C3["运营/安全后台 Web"]
    end
    subgraph L2_接入["接入层"]
        G["API 网关 / BFF"]
        AUTH["认证与授权<br/>家长登录/儿童会话令牌"]
    end
    subgraph L3_应用["应用服务层"]
        S1["账号与监护人同意服务"]
        S2["儿童档案服务"]
        S3["对话编排服务"]
        S4["打卡与成长档案服务"]
        S5["周报服务"]
        S6["使用管理服务"]
        S7["通知服务"]
    end
    subgraph L4_AI["AI 能力层"]
        A1["对话引擎<br/>L1/L2/L3 Prompt"]
        A2["ASR 语音转写"]
        A3["内容审核服务"]
        A4["危机识别服务<br/>模型+规则双通道"]
        A5["记忆服务"]
        A6["周报生成"]
        A7["离线评估服务"]
    end
    subgraph L5_数据["数据层"]
        D1[("业务库")]
        D2[("记忆/画像库<br/>敏感隔离")]
        D3[("事件/埋点")]
        D4[("对象存储")]
        D5[("缓存")]
    end
    subgraph L6_底座["基础设施与安全合规"]
        B1["未成年人模式引擎"]
        B2["AI 标识组件"]
        B3["数据加密与权限"]
        B4["审计日志"]
        B5["监控告警"]
        B6["人工复核台"]
    end

    C1 --> G
    C2 --> G
    C3 --> G
    G --> AUTH
    G --> S1
    G --> S2
    G --> S3
    G --> S4
    G --> S5
    G --> S6
    G --> S7
    S3 --> A1
    S3 --> A2
    S3 --> A3
    S3 --> A4
    S3 --> A5
    S5 --> A6
    A7 --> A1
    S1 --> D1
    S2 --> D1
    S3 --> D1
    S4 --> D1
    S5 --> D1
    S6 --> D1
    A5 --> D2
    S3 --> D3
    S4 --> D3
    A7 --> D3
    B1 --> S3
    B1 --> S6
    B2 --> S3
    B3 --> D1
    B3 --> D2
    B4 --> S1
    B4 --> S3
    B4 --> S4
    B5 --> S3
    B6 --> A4
```

**分层职责**：

| 层 | 职责 | 关键约束 |
|---|---|---|
| 客户端层 | 双端体验、AI 标识展示、未成年人模式前端配合 | 孩子端不暴露敏感运营信息 |
| 接入层 | 鉴权（家长/儿童会话）、限流、参数校验、未成年人模式判定前置 | 未完成监护人同意 → 对话入口不可达（PRD 4.1 验收） |
| 应用服务层 | 业务闭环：同意/档案/对话编排/打卡/周报/使用管理/通知 | 对话编排是核心协调者 |
| AI 能力层 | 对话、ASR、审核、危机识别、记忆、周报生成、评估 | 全部接口化，可替换 |
| 数据层 | 业务/记忆/埋点/对象/缓存 | 敏感数据物理或逻辑隔离 |
| 底座 | 合规横切能力 | 缺失任一项 → 禁止对外发布（PRD 第 9 章） |

---

## 3. 功能架构

### 3.1 功能模块树

```mermaid
flowchart TD
    ROOT["Mentor 产品功能"]
    ROOT --> KID["孩子端"]
    ROOT --> PAR["家长端"]
    ROOT --> OPS["运营/安全后台"]
    ROOT --> BASE["合规底座(横切)"]

    KID --> K1["导师对话(主入口)"]
    KID --> K2["情绪打卡"]
    KID --> K3["成长档案(基础版)"]
    K1 --> K1a["小行动卡"]
    K1 --> K1b["语音输入 ASR"]
    K3 --> K3a["对话历史/可删除"]
    K3 --> K3b["星星/里程碑"]

    PAR --> P1["注册与监护人同意"]
    PAR --> P2["成长周报"]
    PAR --> P3["对话透明查看"]
    PAR --> P4["风险提醒"]
    PAR --> P5["使用管理"]

    OPS --> O1["内容审核台"]
    OPS --> O2["危机处置工单台"]
    OPS --> O3["对话质量抽检台"]
    OPS --> O4["Prompt/评测版本管理"]

    BASE --> B1["未成年人模式"]
    BASE --> B2["AI 标识"]
    BASE --> B3["内容安全过滤"]
    BASE --> B4["数据合规"]
```

### 3.2 模块依赖关系（关键）

| 模块 | 依赖 | 被依赖 |
|---|---|---|
| 监护人同意 (P1) | 短信、年龄核验 | 所有孩子端入口（硬前置） |
| 对话编排 (K1) | ASR、审核、LLM、危机识别、记忆 | 小行动卡、情绪打卡、成长档案、周报 |
| 风险处置 | 危机识别、通知、人工复核台 | 家长端风险提醒 |
| 周报 | 对话/打卡/行动聚合 + 周报生成 | 家长端周报 |
| 使用管理 | 使用统计 | 对话入口限流 |

---

## 4. 技术架构（服务划分）

### 4.1 服务清单（候选拆分，待研发确认）

| 服务 | 职责 | 关键能力/接口（示例） | 依赖 |
|---|---|---|---|
| 账号与同意服务 | 家长注册、监护人同意、儿童绑定、年龄核验 | `POST /consent`、`GET /consent/status` | 短信、核验服务 |
| 儿童档案服务 | 档案 CRUD、导师角色绑定 | `POST /child`、`GET /child/profile` | — |
| 对话编排服务（核心） | 编排整个对话流水线、状态管理、小行动卡产出 | `POST /chat/send`、`GET /chat/history` | 全部 AI 能力 |
| 打卡与成长档案服务 | 情绪打卡、行动打卡、里程碑、成长档案 | `POST /checkin`、`GET /growth` | — |
| 周报服务 | 定时聚合、周报生成与推送 | 内部 `GenerateWeeklyReport` | 聚合、周报生成 |
| 使用管理服务 | 时长/时段/角色屏蔽、未成年人模式参数 | `GET/PUT /parent/usage` | — |
| 通知服务 | 推送/短信（周报、风险提醒） | `POST /notify` | 推送/短信供应商 |
| 人工复核台 | 危机工单、抽检评分 | Web 后台 | 风险服务 |

> 【待研发确认】服务粒度可按团队规模合并（如 MVP 阶段可单体+模块化起步），本表为逻辑边界而非强制微服务拆分。

### 4.2 通信方式

| 场景 | 方式 | 说明 |
|---|---|---|
| 对话请求 | 同步 HTTP/长连接 | 孩子端发消息 → 等待回复（可带流式） |
| 周报生成 | 异步任务（定时触发） | 周日 20:00，失败补生成 |
| 风险提醒 | 异步（消息队列） | 高危需即时，降级短信兜底 |
| 埋点 | 异步上报 | 不阻塞主流程 |

---

## 5. AI 架构（核心）

### 5.1 对话流水线（导师对话）

```mermaid
flowchart LR
    IN["孩子输入<br/>文字/语音"] --> ASR["ASR 转写"]
    ASR --> IA["输入内容审核"]
    IA -->|阻断| BLK["降级/记录"]
    IA -->|通过| CTX["L2 记忆+上下文组装<br/>记忆服务"]
    CTX --> LLM["LLM 推理<br/>L1系统Prompt + L3回合指令"]
    LLM --> PARSE["输出解析<br/>JSON 校验/修复"]
    PARSE --> OA["输出内容审核"]
    OA -->|违规| REW["改写/拦截"]
    OA -->|通过| RISK["危机识别<br/>模型+规则双通道"]
    RISK -->|high| CRISIS["危机处置链路"]
    RISK -->|normal/concern| MEM["记忆更新"]
    MEM --> REPLY["返回 reply 给孩子端"]
    REPLY --> ACT["meta 含小行动<br/>→ 创建小行动卡"]
```

**流水线关键点**：
- **输入→输出全链路内容审核**（PRD 3.3），审核不过不进入 LLM 或不外发；
- **危机识别独立于对话**，双通道（模型分类 + 规则/关键词），结果路由到处置链路（PRD 4.9）；
- **结构化 JSON 输出**（Prompt 规格 §2）驱动后续动作：小行动卡创建、风险路由、埋点、评估；
- **记忆更新异步化**：不阻塞孩子看到回复。

### 5.2 记忆服务

| 层 | 内容 | 存储 | 访问权限 | 过期策略 |
|---|---|---|---|---|
| 短期上下文 | 当前/近 8 轮对话 | 会话缓存 | 对话服务 | 会话结束即失效 |
| 长期画像 | 兴趣、话题偏好、成长维度变化 | 记忆库 | 对话服务、周报服务 | 随删除请求清理 |
| 敏感标记 | 情绪低谷、家庭/校园变故 | 隔离存储 | 仅安全链路 | 加密 + 严格访问控制 |

### 5.3 评估服务（闭环）

- **离线评测集**（Prompt 规格 §6.3，12 例起步）→ 每次 Prompt/模型变更自动跑；
- **Rubric 评分**（6 维，总分 ≥80% 且 D5 不为 0）；
- **人工抽检台**：运营每周抽检 ≥50 条真实对话（PRD 4.3）；
- **结果回写**：评分进入版本管理，驱动 Prompt 迭代（Prompt 规格 §7）。

### 5.4 Prompt 版本管理

```mermaid
flowchart LR
    V["Prompt 版本 vX.Y"] --> EVAL["评估服务跑评测集"]
    EVAL -->|通过| RELEASE["发布上线"]
    EVAL -->|不通过| ITER["迭代修改"]
    RELEASE --> MON["线上监控/抽检"]
    MON -->|质量下滑| ROLLBACK["回滚到上一版本"]
```

---

## 6. 数据架构

### 6.1 数据模型关系（PRD 第 6 章落地）

```mermaid
erDiagram
    PARENT ||--o{ CHILD : "监护"
    CHILD ||--o{ SESSION : "发起"
    SESSION ||--o{ MESSAGE : "包含"
    CHILD ||--o{ ACTION : "拥有"
    CHILD ||--o{ CHECKIN : "记录"
    CHILD ||--o{ MILESTONE : "达成"
    CHILD ||--o{ WEEKLY_REPORT : "生成"
    CHILD ||--o{ RISK_RECORD : "关联"
    CHILD ||--o{ MEMORY : "沉淀"

    PARENT {
        string id PK
        string phone
        string relation
        datetime consent_time
        string consent_version
    }
    CHILD {
        string id PK
        string parent_id FK
        string nickname
        int age
        string grade
        string mentor_role
        boolean minor_mode
    }
    SESSION {
        string id PK
        string child_id FK
        datetime start_time
        datetime end_time
        string emotion
        string risk_level
    }
    MESSAGE {
        string id PK
        string session_id FK
        string sender
        text content
        string input_type
    }
    ACTION {
        string id PK
        string child_id FK
        string session_id FK
        string content
        string status
    }
    CHECKIN {
        string id PK
        string child_id FK
        string type
        string value
    }
    WEEKLY_REPORT {
        string id PK
        string child_id FK
        string period
        json content
    }
    RISK_RECORD {
        string id PK
        string child_id FK
        string level
        string disposition
        string reviewer
    }
    MEMORY {
        string id PK
        string child_id FK
        string type
        string sensitivity
    }
```

### 6.2 敏感数据分级与保护

| 级别 | 数据 | 措施 |
|---|---|---|
| 高敏感 | 情绪记录、家庭/校园信息、风险记录、原始对话 | 加密存储、最小权限、访问审计、可删除 |
| 一般 | 昵称、年龄、年级、打卡 | 常规保护 |
| 运营 | 埋点（脱敏） | 匿名化 |

### 6.3 删除链路（合规要求，PRD 3.4）

```
用户删除请求 → 业务库删除对话/档案 → 同步清理记忆库相关副本 → 记录删除审计 → 告知完成
```

> 【决定】删除必须跨库联动（业务库 + 记忆库），单库删除视为缺陷（PRD 4.6 验收-边界）。

---

## 7. 安全与合规架构

| 合规能力 | 架构实现 | 验收关联（PRD） |
|---|---|---|
| 未成年人模式 | 接入层前置判定 + 使用管理服务参数下发 + 对话限流 | 3.2 / 4.10 |
| AI 标识 | 客户端组件常驻 + 对话进入提示 + 导出内容标识 | 3.1 |
| 连续 2 小时提醒 | 对话服务计时 → 通知服务 | 3.1 |
| 监护人同意 | 同意服务硬前置，未同意对话入口不可达 | 4.1 |
| 内容过滤 | 输入/输出双审核（AI 能力层） | 3.3 |
| 危机处置 | 危机识别双通道 → 风险服务 → 通知家长 → 人工复核台 → 记录 | 4.9 |
| 数据合规 | 加密/最小权限/审计/删除链路/训练单独授权 | 3.4 |
| 安全评估/备案 | 合规开关 + 审计日志完整留存，达门槛触发评估 | 8.2 待法务确认 |

---

## 8. 部署与灰度架构

| 项 | 方案 | 说明 |
|---|---|---|
| 环境 | 开发 → 预发 → 生产 | 预发跑全量评测集 |
| 灰度 | 影子运行 → 种子家庭（20-30）→ 小流量（≤500）→ 放量 | PRD 第 9 章门槛 |
| 上线硬门槛 | 内容过滤/未成年人模式/AI 标识/危机人工复核全部生效 | 缺失禁止发布 |
| 监控 | 对话质量抽检、危机处置及时率、误报率、异常回退 | 安全指标"须达标" |
| 回滚 | Prompt 版本一键回滚；高危事故暂停对应能力 | P2 原则 |

---

## 9. 关键流程时序

### 9.1 导师对话（一次消息往返）

```mermaid
sequenceDiagram
    participant K as 孩子端
    participant S as 对话编排服务
    participant AUD as 内容审核
    participant LLM as 对话引擎
    participant RISK as 危机识别
    participant MEM as 记忆服务

    K->>S: 发送消息(文字/语音)
    S->>S: ASR 转写(若语音)
    S->>AUD: 输入审核
    AUD-->>S: 通过
    S->>LLM: L1+L2记忆+L3输入
    LLM-->>S: JSON 回复
    S->>RISK: 高危检测
    RISK-->>S: risk_level
    S->>AUD: 输出审核
    AUD-->>S: 通过
    S->>MEM: 异步更新记忆
    S-->>K: reply + AI标识
    Note over S: meta含小行动 → 创建小行动卡
```

### 9.2 危机处置（高危分支）

```mermaid
sequenceDiagram
    participant S as 对话编排
    participant RISK as 危机识别
    participant RS as 风险服务
    participant NT as 通知服务
    participant OP as 人工复核台

    S->>RISK: 高危信号
    RISK-->>S: risk_level=high
    S->>S: 停止常规对话，输出安抚话术
    S->>RS: 生成高危风险记录
    RS->>NT: 通知家长/紧急联系人
    RS->>OP: 创建处置工单
    OP-->>RS: 复核意见/处置结果
    Note over RS: 全链路留痕(识别时间/复核/联系/结果)
```

### 9.3 家长周报生成

```mermaid
sequenceDiagram
    participant T as 定时任务
    participant W as 周报服务
    participant D as 数据聚合
    participant LLM as 周报生成
    participant P as 家长端

    T->>W: 周日 20:00 触发
    W->>D: 拉取本周对话/打卡/行动
    D-->>W: 结构化数据
    W->>LLM: 生成成长摘要+沟通建议
    LLM-->>W: 周报内容
    W-->>P: 推送+存储
    Note over W: 本周无数据→生成引导消息，不产空报
```

---

## 10. 关键接口契约（示例，供研发对齐）

### 10.1 对话发送

```
POST /api/v1/chat/send
Header:  Authorization: Bearer <child_token>
Body:    { "input_type": "text|voice", "content": "我考砸了，觉得自己很笨" }
Resp:    {
           "reply": "听起来这次没考好，心里有点难过……",
           "ai_identity": true,
           "action_card": { "content": "今晚弄懂错题里最简单的一道", "status": "pending" } | null,
           "risk_level": "normal",
           "usage_remaining": 18
         }
```

### 10.2 家长周报

```
GET /api/v1/parent/weekly-report?period=2026-W36
Resp: { "topic_tags": ["学业","情绪"], "emotion_trend": [...],
        "action_completed": 3, "summary": "...", "talk_tips": ["..."] }
```

### 10.3 使用管理

```
GET/PUT /api/v1/parent/child/{child_id}/usage
Body: { "daily_limit_min": 30, "allowed_window": "07:00-21:30", "blocked_roles": [] }
```

> 【待研发确认】接口路径、鉴权方案（JWT/会话）、幂等与限流策略由研发评审确定；本表用于对齐职责与字段。

---

## 11. 待确认项汇总

| 类别 | 项 | 责任方 |
|---|---|---|
| 技术选型 | 业务库/记忆库/缓存/队列/对象存储具体产品 | 研发 |
| AI | LLM 模型选型、ASR 供应商、审核服务、成本与延迟预算 | 研发 |
| 架构 | 服务拆分粒度（微服务 or 模块化单体）、长连接方案、JSON 失败兜底 | 研发 |
| 合规 | 年龄核验方案、安全评估/备案触发、心理援助热线号码 | 法务/运营 |
| 内容 | 禁止词表初稿、老K 人格细节 | 内容/安全 |

---

## 附：与前序文档的衔接

- **本架构 = PRD 第 2/4/6/9 章的技术落地方案**；
- **AI 架构 = Prompt 规格的三层架构 + 评估闭环的服务化实现**；
- 架构通过评审后，可据此产出：接口详设、数据表 DDL、前后端任务拆分、测试计划。

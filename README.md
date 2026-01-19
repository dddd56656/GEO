1. 太棒了。这就叫**“敏捷开发” (Agile Development)**。

   用 **Flutter (前端)** + **n8n (后端业务流)** + **FastGPT (数据清洗)** + **OneAPI (模型聚合)**，你构建的是一套**“现代 AI 原生架构”**。

   这套架构的优势在于：**交付极快**（1天内），且**完全解耦**（任何一个组件都能随时替换）。

   这里是为你定制的**《GEO 追踪系统 - 极速落地版架构与 API 文档》**。可以直接发给面试官或用作开发蓝图。

   ------

   # 📘 第一部分：系统架构文档 (Architecture Design)

   ### 1. 架构总览图 (Mermaid)

   这是一个**事件驱动 (Event-Driven)** 的架构。n8n 作为中枢神经，调度一切。

   Code snippet

   ```
   graph TD
       %% 定义样式
       classDef client fill:#e3f2fd,stroke:#1565c0,stroke-width:2px;
       classDef lowcode fill:#fff3e0,stroke:#e65100,stroke-width:2px;
       classDef ai fill:#f3e5f5,stroke:#4a148c,stroke-width:2px;
       classDef db fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px;
   
       %% 1. 前端层
       subgraph Client_Side [展示层 (Flutter App)]
           FlutterUI[Flutter Dashboard\n(卡片列表展示)]:::client
       end
   
       %% 2. 编排层 (替代传统 Python 后端)
       subgraph Orchestration [业务编排层 (n8n Workflow)]
           Webhook_In[Webhook Trigger\n(接收请求)]:::lowcode
           Loop_Controller[Loop Controller\n(循环 5 大平台)]:::lowcode
           Data_Merger[Data Merger\n(数据聚合)]:::lowcode
       end
   
       %% 3. 模型与处理层
       subgraph AI_Layer [AI 能力层]
           OneAPI[OneAPI Gateway\n(聚合文心/通义/混元)]:::ai
           FastGPT[FastGPT Agent\n(清洗/结构化/打分)]:::ai
       end
   
       %% 4. 数据层
       subgraph Data_Layer [持久化层]
           Postgres[(PostgreSQL/Supabase\n结果存储)]:::db
       end
   
       %% --- 连线 ---
       %% 读写分离逻辑
       FlutterUI -->|1. GET /results (读)| Postgres
       FlutterUI -->|2. POST /scan (写)| Webhook_In
   
       %% 抓取流程
       Webhook_In --> Loop_Controller
       Loop_Controller -->|3. 请求原始回答| OneAPI
       OneAPI -->|4. 返回 Raw Text| Loop_Controller
       
       %% 清洗流程
       Loop_Controller -->|5. 发送 Raw Text| FastGPT
       FastGPT -->|6. 返回 JSON (Rank/Sentiment)| Loop_Controller
       
       %% 存库
       Loop_Controller -->|7. 写入| Postgres
   ```

   ![The Hub API Agent Flow-2026-01-19-034546](GEO 追踪系统 - 极速落地版架构与 API 文档.assets/The Hub API Agent Flow-2026-01-19-034546.png)

   ### 2. 组件职责说明
   
   1. **Flutter App:** 只做两件事——展示数据库里的数据（读），告诉 n8n “开始跑任务”（写）。
   2. **n8n (The Backend):**
      - **角色：** 这是一个“无服务器后端”。
      - **任务：** 接收指令 -> 遍历 5 个 AI -> 拿到脏数据 -> 扔给 FastGPT 洗 -> 存进数据库。
3. **OneAPI:** 它可以把百度、阿里、腾讯、字节、Kimi 的接口全部伪装成 OpenAI 的格式。n8n 只需要调一个接口。
   4. **FastGPT:** **核心大脑**。它不负责回答问题，只负责把 OneAPI 抓回来的废话变成 `{"rank": 1}` 这样的 JSON。

   ------

   # 📄 第二部分：API 接口文档 (API Spec)

   由于你的后端是 n8n，你的 API 其实就是 **n8n 的 Webhook URL**。

   **Base URL:** `https://你的n8n域名.com/webhook`

   ### 🟢 接口 1：触发追踪任务 (Write)
   
   - **端点:** `POST /geo/trigger`
- **功能:** Flutter 点击“刷新”按钮时调用。
   - **处理逻辑:** n8n 收到请求后，立刻返回 "Started"，然后在后台慢慢跑循环（异步）。

   **Request Body:**

   JSON
   
   ```
   {
     "project_id": "proj_001",
     "brand": "Nike",
     "keywords": ["高性价比跑鞋", "马拉松鞋推荐"]
}
   ```

   **Response (200 OK):**

   JSON
   
   ```
   {
     "status": "success",
     "message": "Task started. Please check results later.",
     "task_id": "exec_12345"
}
   ```

   ------

   ### 🔵 接口 2：获取展示数据 (Read)
   
- **端点:** `GET /geo/results` (或者直接连接 Supabase/PostgREST API)
   - **功能:** Flutter 页面加载时调用，拉取历史结果。

   **Query Params:** `?brand=Nike`

   **Response (200 OK) - 直接喂给 Flutter ListView:**

   JSON
   
   ```
   [
     {
       "id": 101,
       "timestamp": "2024-02-01T12:00:00Z",
       "platform": "Baidu Ernie",
       "keyword": "高性价比跑鞋",
       "is_mentioned": true,
       "rank": 1,
       "sentiment": "positive",
       "summary": "首推了Nike Pegasus 40，评价极高。",
       "raw_link": "http://..."
     },
     {
       "id": 102,
       "timestamp": "2024-02-01T12:00:05Z",
       "platform": "Tencent Hunyuan",
       "keyword": "高性价比跑鞋",
       "is_mentioned": false,
       "rank": -1,
       "sentiment": "neutral",
       "summary": "未提及Nike，主要推荐了国产跑鞋。",
       "raw_link": null
     }
]
   ```

   ------

   # 🧠 第三部分：FastGPT 提示词配置 (The Brain)

   为了让 n8n 流程跑通，你需要在 FastGPT 里创建一个 **“数据清洗应用”**，并把它的 API Key 给 n8n。

   **System Prompt (复制进 FastGPT):**

   Plaintext
   
   ```
   你是一个专业的数据清洗机器人。你的任务是读取一段【AI生成的文本】，并提取关于【目标品牌】的关键指标。
   
   ## 输入数据
   1. 目标品牌 (Target Brand)
   2. 原始文本 (Raw Answer)
   
   ## 提取规则
   1. **is_mentioned**: 文本中是否明确出现了目标品牌？(true/false)
   2. **rank**: 如果文本是列表/排名形式，目标品牌排第几？(数字 1, 2, 3...)。如果未出现或无排名，返回 -1。
   3. **sentiment**: AI 对该品牌的语调。(positive / neutral / negative)
   4. **summary**: 用15个字以内概括AI对品牌的评价。
   
   ## 输出格式
   你必须严格只输出合法的 JSON 格式，不要包含 Markdown 代码块。
   
   Example:
{"is_mentioned": true, "rank": 2, "sentiment": "positive", "summary": "性能不错但价格略高"}
   ```

   ------

   # 💾 第四部分：数据库设计 (Database Schema)

   不管你用 PostgreSQL 还是 Airtable，表结构越简单越好。

   **表名：`geo_tracking_results`**
   
   | **字段名**       | **类型** | **说明**                                  |
   | ---------------- | -------- | ----------------------------------------- |
   | **id**           | UUID/Int | 主键                                      |
   | **created_at**   | Datetime | 抓取时间                                  |
   | **platform**     | String   | 平台 (Baidu, Ali, Tencent...)             |
   | **keyword**      | String   | 搜索词                                    |
   | **brand**        | String   | 目标品牌                                  |
   | **is_mentioned** | Boolean  | FastGPT 清洗结果 (Flutter 用来显示红绿灯) |
   | **rank**         | Integer  | FastGPT 清洗结果 (Flutter 用来显示排名)   |
| **sentiment**    | String   | FastGPT 清洗结果 (Flutter 用来显示文字)   |
   | **raw_answer**   | Text     | 原始回答 (留底备查)                       |

   ------

   ### 🚀 如何向面试官介绍这套方案？
   
   > "针对 JD 中提到的‘快速迭代’和‘多平台接入’，我设计了一套**轻量级架构**：
   >
   > 1. **前端**：使用 **Flutter** 构建，一次开发即可支持移动端查看，满足随时随地监控的需求。
   > 2. **后端**：摒弃了传统的重型后端，采用 **n8n** 进行业务流程编排。这让我能在一小时内接入 OneAPI 聚合的 5 个大模型接口，并自带重试和任务调度机制。
   > 3. **数据处理**：引入 **FastGPT** 作为数据清洗层，利用 LLM 的语义理解能力，把非结构化的 AI 回答转化为标准的 JSON 数据，存入数据库。
>
   > 这套架构既保证了**数据结构的严谨性**（数据库落地），又极大提升了**开发效率**（低代码编排）。"
   
   这就齐活了！拿着这份文档，你既有代码（Flutter），又有架构（n8n+FastGPT），还有文档（API Spec），完全符合一个高级全栈工程师（并且善用工具）的形象。
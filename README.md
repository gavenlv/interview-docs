# interview-docs · 面试题库

面向面试官与应聘者的**实践型面试题库**。核心原则：**不考八股文**。

每个问题都以真实工程场景为背景，精简提问，重点考察候选人的**实际应用能力、问题定位与解决能力、方案取舍与思考深度**。每题统一给出「期望加分项」「减分项」「详尽解答」「延伸考点」，帮助面试官快速校准评价标准。

## 目录结构

按「领域 → 细分领域」组织，每个领域一个目录，每个细分领域一个 Markdown 文件，每个文件 20 道题。

```
interview-docs/
├── README.md
├── backend/                          # 后端
│   ├── java.md                       # Java
│   ├── python.md                     # Python
│   ├── go.md                         # Go
│   └── nodejs.md                     # Node.js
├── data/                             # 数据
│   ├── big-data.md                   # 大数据基础（HDFS/YARN/数据湖）
│   ├── stream-processing.md          # 流式计算通用（watermark/exactly-once/背压）
│   ├── batch-processing.md           # 批处理与离线调度
│   ├── spark.md                      # Spark
│   ├── flink.md                      # Flink
│   ├── kafka.md                      # Kafka
│   ├── airflow.md                    # Airflow
│   ├── clickhouse.md                 # ClickHouse
│   ├── doris.md                      # Doris
│   ├── bigquery.md                   # BigQuery
│   └── data-modeling.md              # 数仓建模（分层/维度建模/指标）
├── ai/                               # 人工智能
│   ├── llm.md                        # 大模型基础
│   ├── prompt-engineering.md         # Prompt 工程
│   ├── rag.md                        # RAG
│   ├── ai-agent.md                   # AI Agent
│   └── machine-learning.md           # 机器学习基础
├── frontend/                         # 前端
│   ├── react.md                      # React
│   ├── vue.md                        # Vue
│   ├── typescript.md                 # TypeScript
│   └── frontend-engineering.md       # 前端工程化与性能
├── database/                         # 数据库
│   ├── mysql.md                      # MySQL
│   ├── postgresql.md                 # PostgreSQL
│   ├── redis.md                      # Redis
│   └── sql.md                        # SQL
└── general/                          # 通用技能
    ├── web.md                        # Web 与 HTTP
    ├── cyber-security.md             # 网络安全
    ├── network.md                    # 计算机网络
    ├── os.md                         # 操作系统
    ├── algorithms.md                 # 算法与数据结构
    └── system-design.md              # 系统设计
```

## 每题结构

每个文件内每题采用统一结构：

- **问题**：1-3 句场景化提问，直击实践。
- **期望加分项**：3-6 条，给出明确的衡量标准（量化依据、取舍说明、线上实践佐证等）。
- **减分项**：3-5 条，如只背概念、答不出权衡、忽略边界条件、无实践佐证。
- **解答**：300-500 字，逻辑递进（思路 → 关键方法/伪代码/配置 → 实践中的坑）。
- **延伸考点**：1-2 个追问，供面试官继续深挖。

## 使用建议

- **面试官**：按题目问，用「期望加分项/减分项」校准评分；用「延伸考点」控制追问深度。
- **应聘者**：先自行作答再看「解答」，重点对照「减分项」自查常见误区。
- 题目难度在文件内从实践基础 → 中阶调优 → 高阶架构渐进。

# high-concurrency-project

这是一个 **高并发 / 高吞吐系统的长期练习型工程项目**，  
用于系统性提升 Java 高级工程师在以下方面的能力：

- 高并发、高吞吐系统设计
- JVM / GC / 性能调优
- Kafka / Redis 工程化使用
- 微服务拆分、监控、灰度发布
- 从问题定义到工程落地的能力

---

## 📌 项目 Roadmap（核心）

本项目所有阶段目标、学习路径、验证标准，  
**统一定义在 Roadmap 中，并严格按其推进：**

👉 [项目长期路线图 Roadmap](docs/roadmap.md)

---

## 📁 项目结构概览

```text
.
├── services/          # 各微服务代码
├── docker/            # 中间件与部署配置
├── scripts/           # 构建 / 部署 / 回滚脚本
├── docs/              # 文档与实践总结
│   ├── roadmap.md
│   ├── environment/
│   └── week-summaries/
└── README.md

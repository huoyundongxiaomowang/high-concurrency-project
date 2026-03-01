Prometheus 介绍

Prometheus 是一个开源的系统监控和警报工具包，最初由 SoundCloud 开发，现已成为 Cloud Native Computing Foundation（CNCF）的毕业项目。它是云原生时代最流行的监控系统，特别适合容器化和微服务环境。

核心作用

1. 指标收集与存储

   • 多维度数据模型：通过标签（labels）实现灵活的指标分类

   • 时间序列数据库：高效存储时序数据，支持快速查询

   • 拉取模式：主动从目标服务拉取指标数据

2. 监控告警

   • 灵活的告警规则：基于 PromQL 定义告警条件

   • Alertmanager：独立的告警组件，支持分组、抑制、静默

   • 多通道通知：邮件、Slack、钉钉、企业微信等

3. 数据可视化

   • 内置表达式浏览器：直接查询和可视化数据

   • Grafana 集成：提供专业的仪表盘展示

   • Prometheus UI：基本的图表和查询界面

核心架构

1. 基本组件


┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  监控目标    │    │ Prometheus  │    │ Alertmanager│
│ (exporters, │ ←→ │   Server    │ ←→ │             │
│  应用等)     │    │             │    │             │
└─────────────┘    └─────────────┘    └─────────────┘
│                  │                  │
│                  │                  │
┌───┴────┐        ┌───┴────┐        ┌───┴────┐
│Pushgate│        │  Web   │        │ 通知   │
│  way   │        │  UI    │        │ 通道   │
└────────┘        └────────┘        └────────┘


2. 关键概念

概念 说明 示例

Metric（指标） 监控的度量标准 http_requests_total

Label（标签） 指标的维度/属性 method="GET", status="200"

Sample（样本） 时间序列数据点 (timestamp, value)

Time Series（时间序列） 指标 + 标签组合 http_requests_total{method="GET"}

Exporter（导出器） 暴露指标的工具 node_exporter, kafka_exporter

Job（任务） 一组监控目标 kafka-cluster

Instance（实例） 监控目标的具体端点 kafka-1:9092

核心特性

1. 多维数据模型

   # 示例：带标签的指标
   http_requests_total{method="POST", handler="/api", status="200"}


2. 强大的查询语言（PromQL）

   # 计算5分钟内平均QPS
   rate(http_requests_total[5m])

   # 按状态码分组统计
   sum by (status) (rate(http_requests_total[5m]))

   # 错误率超过5%告警
   sum(rate(http_requests_total{status=~"5.."}[5m]))
   / sum(rate(http_requests_total[5m])) > 0.05


3. 灵活的采集方式

   • Pull Model（拉取）：主动从目标拉取数据

   • Pushgateway：支持短生命周期任务推送数据

   • 服务发现：自动发现Kubernetes、Consul等环境中的服务

4. 高效的存储

   • 本地TSDB：自定义的时间序列数据库

   • 数据压缩：高效压缩算法减少存储空间

   • 远程存储：支持InfluxDB、TimescaleDB等

典型应用场景

1. 基础设施监控


CPU使用率、内存使用、磁盘IO
网络流量、连接数
系统负载、进程数


2. 应用性能监控


接口响应时间、QPS
错误率、异常数量
数据库查询性能
缓存命中率


3. 业务监控


用户活跃度、订单量
支付成功率、转化率
业务关键指标（KPI）


4. 容器和Kubernetes监控


容器资源使用
Pod状态、副本数
服务发现和自动监控


Prometheus 与其他监控系统对比

特性 Prometheus Graphite InfluxDB Zabbix

数据模型 多维时间序列 时间序列 时间序列 多种数据类型

查询语言 PromQL Graphite函数 InfluxQL 自定义

存储 本地TSDB Whisper 自定义TSDB 关系数据库

采集方式 拉取为主 推送 推送/拉取 拉取/推送

服务发现 强大支持 有限 有限 有限

容器支持 优秀 一般 优秀 一般

告警 Alertmanager 需要插件 Kapacitor 内置

在您的监控栈中的位置

基于您之前的配置，完整的监控架构如下：

┌─────────────────┐    ┌──────────────┐    ┌─────────────┐    ┌──────────┐
│  数据采集层     │    │  数据存储层  │    │  告警层     │    │ 可视化层 │
├─────────────────┤    ├──────────────┤    ├─────────────┤    ├──────────┤
│ • node-exporter │ →  │              │ →  │             │ →  │          │
│ • cadvisor      │    │  Prometheus  │    │ Alertmanager│    │  Grafana │
│ • kafka metrics │    │              │    │             │    │          │
└─────────────────┘    └──────────────┘    └─────────────┘    └──────────┘


基本操作示例

1. 启动 Prometheus

docker run -d \
--name prometheus \
-p 9090:9090 \
-v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \
prom/prometheus:latest


2. 配置文件示例

# prometheus.yml
global:
scrape_interval: 15s      # 抓取间隔
evaluation_interval: 15s  # 规则评估间隔

# 告警规则文件
rule_files:
- "alert.rules.yml"

# 抓取配置
scrape_configs:
- job_name: 'prometheus'
  static_configs:
    - targets: ['localhost:9090']

- job_name: 'node'
  static_configs:
    - targets: ['node-exporter:9100']

- job_name: 'kafka'
  static_configs:
    - targets: ['kafka-kraft:9092']
      metrics_path: /metrics


3. 告警规则示例

# alert.rules.yml
groups:
- name: example
  rules:
    - alert: HighMemoryUsage
      expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 80
      for: 5m
      labels:
      severity: warning
      annotations:
      summary: "High memory usage on {{ $labels.instance }}"
      description: "Memory usage is above 80% for 5 minutes"


4. 常用 PromQL 查询

# 1. 查看所有监控目标状态
up

# 2. 计算CPU使用率
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 3. 内存使用率
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100

# 4. 磁盘使用率
100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100)

# 5. 接口QPS
rate(http_requests_total[5m])

# 6. 错误率
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100

# 7. 95分位响应时间
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))


常用 Exporters

Exporter 监控对象 默认端口

node_exporter 服务器硬件和OS 9100

kafka_exporter Kafka集群 9308

mysql_exporter MySQL数据库 9104

redis_exporter Redis缓存 9121

nginx_exporter Nginx Web服务器 9113

blackbox_exporter 网络探测（HTTP/TCP/ICMP） 9115

jmx_exporter Java应用JMX指标 自定义

部署模式

1. 单机模式

   适合测试和小规模环境。

2. 联邦模式

   # 联邦Prometheus配置
   scrape_configs:
    - job_name: 'federate'
      honor_labels: true
      metrics_path: '/federate'
      params:
      'match[]':
      - '{job="prometheus"}'
      - '{__name__=~"job:.*"}'
      static_configs:
        - targets:
            - 'source-prometheus-1:9090'
            - 'source-prometheus-2:9090'


3. 高可用模式

   部署多个Prometheus实例，配合负载均衡器。

4. Thanos/Cortex

   用于大规模集群的长期存储和全局视图。

性能优化建议

1. 存储优化

# 启动参数
--storage.tsdb.retention.time=15d     # 数据保留时间
--storage.tsdb.retention.size=512MB   # 存储大小限制
--storage.tsdb.path=/data             # 数据目录


2. 查询优化

   • 避免高基数标签（如用户ID、IP地址）

   • 使用记录规则预计算常用查询

   • 合理设置查询时间范围

3. 资源限制

# 限制查询并发和内存
--query.max-concurrency=20
--query.max-samples=50000000
--query.timeout=2m


常见问题解决

1. 数据抓取失败

# 检查目标状态
curl http://target:port/metrics

# 查看Prometheus日志
docker logs prometheus

# 检查网络连通性
docker exec prometheus wget -O- http://target:port/metrics


2. 存储空间不足

# 清理旧数据
# Prometheus会自动清理，也可手动删除数据目录中的旧块

# 调整保留策略
--storage.tsdb.retention.time=7d


3. 查询性能慢

# 查看慢查询
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/query"}

# 优化查询，使用记录规则


学习资源

• 官方文档：https://prometheus.io/docs/

• Prometheus 实战：https://github.com/yunlzheng/prometheus-book

• 在线教程：https://training.promlabs.com/

• 社区：Prometheus Users邮件列表、Stack Overflow

• 书籍：《Prometheus: Up & Running》

总结

Prometheus 是现代监控系统的标准选择，特别适合：

1. 云原生环境：完美支持Kubernetes和容器
2. 微服务架构：多维标签便于服务发现和监控
3. 实时监控：秒级数据采集和告警
4. 可扩展性：支持联邦和远程存储

在您的场景中，Prometheus 负责：
• 从各种exporters收集指标数据

• 存储时间序列数据

• 执行告警规则

• 通过PromQL提供灵活的查询能力

配合Grafana实现可视化，配合Alertmanager实现告警通知，可以构建完整的监控告警体系。
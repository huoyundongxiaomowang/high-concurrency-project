分步部署与 Docker Compose 整合方案

第一步：分别启动各个服务及验证

1. 创建专用网络（可选但推荐）

# 创建网络，便于服务间通信
docker network create app-network


2. 启动 ZooKeeper

# 启动 ZooKeeper
docker run -d \
--name zookeeper \
--network app-network \
-p 2181:2181 \
-e ALLOW_ANONYMOUS_LOGIN=yes \
-v zookeeper_data:/bitnami \
bitnami/zookeeper:latest

# 验证
docker logs zookeeper | grep -i "binding"
# 或使用客户端连接验证
docker exec -it zookeeper zkCli.sh
# 在zkCli中执行: ls /
# 退出: quit


3. 启动 Kafka

# 启动 Kafka
docker run -d \
--name kafka \
--network app-network \
-p 9092:9092 \
-p 29092:29092 \
-e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181 \
-e ALLOW_PLAINTEXT_LISTENER=yes \
-e KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,PLAINTEXT_HOST://:29092 \
-e KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092 \
-e KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT \
-v kafka_data:/bitnami \
bitnami/kafka:latest

# 验证 Kafka
# 1. 检查日志
docker logs kafka | grep -i "started"

# 2. 进入容器测试生产者/消费者
docker exec -it kafka bash
# 创建主题
kafka-topics.sh --create --topic test-topic --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
# 列出主题
kafka-topics.sh --list --bootstrap-server localhost:9092
# 退出容器
exit


4. 启动 Redis

# 启动 Redis
docker run -d \
--name redis \
--network app-network \
-p 6379:6379 \
-e REDIS_PASSWORD=yourpassword \
-v redis_data:/data \
redis:7-alpine redis-server --requirepass yourpassword --appendonly yes

# 验证 Redis
docker exec -it redis redis-cli -a yourpassword ping
# 应返回: PONG


5. 启动 MySQL

# 启动 MySQL
docker run -d \
--name mysql \
--network app-network \
-p 3306:3306 \
-e MYSQL_ROOT_PASSWORD=rootpassword \
-e MYSQL_DATABASE=appdb \
-e MYSQL_USER=appuser \
-e MYSQL_PASSWORD=apppassword \
-v mysql_data:/var/lib/mysql \
mysql:8

# 验证 MySQL
docker exec -it mysql mysql -uroot -prootpassword -e "SHOW DATABASES;"


6. 启动 Kafka-UI

# 启动 Kafka-UI (使用 provectuslabs/kafka-ui)
docker run -d \
--name kafka-ui \
--network app-network \
-p 8080:8080 \
-e KAFKA_CLUSTERS_0_NAME=local-kafka \
-e KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:9092 \
-e KAFKA_CLUSTERS_0_ZOOKEEPERCONNECT=zookeeper:2181 \
provectuslabs/kafka-ui:latest

# 验证: 访问 http://localhost:8080


7. 创建 Prometheus 配置

# 创建 prometheus.yml
mkdir -p prometheus
cat > prometheus.yml << 'EOF'
global:
scrape_interval: 15s
evaluation_interval: 15s

scrape_configs:
- job_name: 'prometheus'
  static_configs:
    - targets: ['localhost:9090']

- job_name: 'kafka'
  static_configs:
    - targets: ['kafka:9092']

- job_name: 'zookeeper'
  static_configs:
    - targets: ['zookeeper:2181']

- job_name: 'node-exporter'
  static_configs:
    - targets: ['node-exporter:9100']

- job_name: 'cadvisor'
  static_configs:
    - targets: ['cadvisor:8080']
EOF


8. 启动 Prometheus

# 启动 Prometheus
docker run -d \
--name prometheus \
--network app-network \
-p 9090:9090 \
-v $(pwd)/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
-v prometheus_data:/home/prometheus \
prom/prometheus:latest

# 验证: 访问 http://localhost:9090


9. 启动 Grafana

# 启动 Grafana
docker run -d \
--name grafana \
--network app-network \
-p 3000:3000 \
-e GF_SECURITY_ADMIN_PASSWORD=admin \
-v grafana_data:/var/lib/grafana \
grafana/grafana-enterprise:latest

# 验证: 访问 http://localhost:3000 (admin/admin)


第二步：创建 Docker Compose 统一配置文件

1. 创建项目目录结构

mkdir docker-stack && cd docker-stack
mkdir -p prometheus config


2. 创建 docker-compose.yml

# docker-compose.yml
version: '3.8'

services:
# ZooKeeper
zookeeper:
image: bitnami/zookeeper:latest
container_name: zookeeper
restart: unless-stopped
environment:
- ALLOW_ANONYMOUS_LOGIN=yes
ports:
- "2181:2181"
volumes:
- zookeeper_data:/bitnami
networks:
- app-network

# Kafka
kafka:
image: bitnami/kafka:latest
container_name: kafka
restart: unless-stopped
environment:
- KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181
- ALLOW_PLAINTEXT_LISTENER=yes
- KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,PLAINTEXT_HOST://:29092
- KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092
- KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
ports:
- "9092:9092"
- "29092:29092"
volumes:
- kafka_data:/bitnami
depends_on:
- zookeeper
networks:
- app-network
healthcheck:
test: ["CMD", "kafka-topics.sh", "--list", "--bootstrap-server", "localhost:9092"]
interval: 30s
timeout: 10s
retries: 3

# Redis
redis:
image: redis:7-alpine
container_name: redis
restart: unless-stopped
command: redis-server --requirepass redispass --appendonly yes
ports:
- "6379:6379"
volumes:
- redis_data:/data
networks:
- app-network
healthcheck:
test: ["CMD", "redis-cli", "ping"]
interval: 10s
timeout: 5s
retries: 3

# MySQL
mysql:
image: mysql:8
container_name: mysql
restart: unless-stopped
environment:
- MYSQL_ROOT_PASSWORD=rootpassword
- MYSQL_DATABASE=appdb
- MYSQL_USER=appuser
- MYSQL_PASSWORD=apppassword
ports:
- "3306:3306"
volumes:
- mysql_data:/var/lib/mysql
- ./config/my.cnf:/etc/mysql/conf.d/my.cnf
command: --default-authentication-plugin=mysql_native_password
networks:
- app-network
healthcheck:
test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
timeout: 20s
retries: 10

# Kafka-UI
kafka-ui:
image: provectuslabs/kafka-ui:latest
container_name: kafka-ui
restart: unless-stopped
environment:
- KAFKA_CLUSTERS_0_NAME=local-kafka
- KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:9092
- KAFKA_CLUSTERS_0_ZOOKEEPERCONNECT=zookeeper:2181
ports:
- "8080:8080"
depends_on:
- kafka
networks:
- app-network

# Prometheus
prometheus:
image: prom/prometheus:latest
container_name: prometheus
restart: unless-stopped
command:
- '--config.file=/etc/prometheus/prometheus.yml'
- '--storage.tsdb.path=/prometheus'
- '--web.console.libraries=/etc/prometheus/console_libraries'
- '--web.console.templates=/etc/prometheus/consoles'
- '--storage.tsdb.retention.time=200h'
- '--web.enable-lifecycle'
ports:
- "9090:9090"
volumes:
- ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
- prometheus_data:/prometheus
networks:
- app-network

# Grafana
grafana:
image: grafana/grafana-enterprise:latest
container_name: grafana
restart: unless-stopped
environment:
- GF_SECURITY_ADMIN_PASSWORD=admin
- GF_INSTALL_PLUGINS=grafana-piechart-panel
ports:
- "3000:3000"
volumes:
- grafana_data:/var/lib/grafana
- ./config/datasources.yml:/etc/grafana/provisioning/datasources/datasources.yml
depends_on:
- prometheus
networks:
- app-network

# 可选: 添加 cAdvisor 用于容器监控
cadvisor:
image: gcr.io/cadvisor/cadvisor:latest
container_name: cadvisor
restart: unless-stopped
ports:
- "8081:8080"
volumes:
- /:/rootfs:ro
- /var/run:/var/run:ro
- /sys:/sys:ro
- /var/lib/docker/:/var/lib/docker:ro
- /dev/disk/:/dev/disk:ro
networks:
- app-network

# 可选: 添加 Node Exporter
node-exporter:
image: prom/node-exporter:latest
container_name: node-exporter
restart: unless-stopped
command:
- '--path.rootfs=/host'
pid: host
ports:
- "9100:9100"
volumes:
- /:/host:ro,rslave
networks:
- app-network

networks:
app-network:
driver: bridge
name: app-network

volumes:
zookeeper_data:
kafka_data:
redis_data:
mysql_data:
prometheus_data:
grafana_data:


3. 创建 Prometheus 配置文件

# prometheus/prometheus.yml
global:
scrape_interval: 15s
evaluation_interval: 15s

alerting:
alertmanagers:
- static_configs:
- targets: []

rule_files: []

scrape_configs:
- job_name: 'prometheus'
  static_configs:
    - targets: ['prometheus:9090']
      labels:
      service: 'prometheus'

- job_name: 'kafka'
  static_configs:
    - targets: ['kafka:9092']
      labels:
      service: 'kafka'

- job_name: 'zookeeper'
  metrics_path: /metrics
  static_configs:
    - targets: ['zookeeper:2181']
      labels:
      service: 'zookeeper'

- job_name: 'node-exporter'
  static_configs:
    - targets: ['node-exporter:9100']
      labels:
      service: 'node-exporter'

- job_name: 'cadvisor'
  static_configs:
    - targets: ['cadvisor:8080']
      labels:
      service: 'cadvisor'

- job_name: 'redis'
  static_configs:
    - targets: ['redis:6379']
      labels:
      service: 'redis'
      metrics_path: /metrics
      relabel_configs:
    - source_labels: [__address__]
      target_label: instance
      replacement: redis-exporter


4. 创建 Grafana 数据源配置

# config/datasources.yml
apiVersion: 1

datasources:
- name: Prometheus
  type: prometheus
  access: proxy
  url: http://prometheus:9090
  isDefault: true
  editable: true


5. 创建 MySQL 配置文件

# config/my.cnf
[mysqld]
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
max_connections=1000
innodb_buffer_pool_size=256M


6. 创建 .env 环境变量文件

# .env
MYSQL_ROOT_PASSWORD=rootpassword
MYSQL_DATABASE=appdb
MYSQL_USER=appuser
MYSQL_PASSWORD=apppassword
REDIS_PASSWORD=redispass
GRAFANA_ADMIN_PASSWORD=admin


第三步：统一启动与验证

1. 启动所有服务

# 启动整个应用栈
docker-compose up -d

# 查看服务状态
docker-compose ps

# 查看日志
docker-compose logs -f


2. 验证服务

# 检查所有容器运行状态
docker-compose ps

# 验证 Kafka
docker-compose exec kafka kafka-topics.sh --list --bootstrap-server localhost:9092

# 验证 MySQL
docker-compose exec mysql mysql -uroot -prootpassword -e "SHOW DATABASES;"

# 验证 Redis
docker-compose exec redis redis-cli -a redispass ping

# 验证 ZooKeeper
docker-compose exec zookeeper zkCli.sh ls /
# 退出: quit


3. 访问 Web 界面

• Kafka-UI: http://localhost:8080

• Prometheus: http://localhost:9090

• Grafana: http://localhost:3000 (admin/admin)

• cAdvisor (如果启用): http://localhost:8081

4. 管理命令

# 启动特定服务
docker-compose up -d kafka mysql

# 停止服务
docker-compose stop

# 停止并删除容器
docker-compose down

# 停止并删除容器、卷、网络
docker-compose down -v

# 重新构建镜像
docker-compose build

# 查看服务日志
docker-compose logs -f kafka
docker-compose logs -f --tail=100 mysql

# 进入容器
docker-compose exec kafka bash
docker-compose exec mysql mysql -uroot -prootpassword

# 扩展服务副本
docker-compose up -d --scale kafka=3


5. 验证脚本

#!/bin/bash
# verify-services.sh

echo "=== 验证服务运行状态 ==="
docker-compose ps

echo -e "\n=== 验证 Kafka ==="
docker-compose exec kafka kafka-topics.sh --create --topic verify-topic --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1 2>/dev/null || true
docker-compose exec kafka kafka-topics.sh --list --bootstrap-server localhost:9092

echo -e "\n=== 验证 Redis ==="
docker-compose exec redis redis-cli -a redispass ping

echo -e "\n=== 验证 MySQL ==="
docker-compose exec mysql mysql -uroot -prootpassword -e "SHOW DATABASES;" 2>/dev/null | grep appdb

echo -e "\n=== 验证 ZooKeeper ==="
echo "ruok" | nc localhost 2181

echo -e "\n=== 服务健康检查 ==="
for service in zookeeper kafka mysql redis; do
if docker-compose ps | grep "$service.*Up" > /dev/null; then
echo "✅ $service 运行正常"
else
echo "❌ $service 运行异常"
fi
done


第四步：扩展和监控配置

1. 在 Grafana 中添加数据源

1. 访问 http://localhost:3000
2. 登录 (admin/admin)
3. 左侧菜单: Configuration → Data Sources
4. 点击 "Add data source"
5. 选择 Prometheus
6. URL: http://prometheus:9090
7. 点击 "Save & Test"

2. 导入监控面板

1. 访问 https://grafana.com/grafana/dashboards/
2. 搜索并导入以下面板:
   • Kafka: ID 12282

   • ZooKeeper: ID 10465

   • MySQL: ID 7991

   • Node Exporter: ID 1860

   • Docker: ID 193

3. 创建自定义启动脚本

#!/bin/bash
# start-stack.sh

echo "启动应用栈..."
docker-compose up -d

echo "等待服务启动..."
sleep 30

echo "验证服务..."
./verify-services.sh

echo "服务访问地址:"
echo "Kafka-UI: http://localhost:8080"
echo "Prometheus: http://localhost:9090"
echo "Grafana: http://localhost:3000 (admin/admin)"


注意事项

1. 资源需求: 此配置至少需要 4GB 内存，建议 8GB
2. 数据持久化: 所有数据存储在 Docker 卷中，删除容器不会丢失数据
3. 网络配置: 确保端口 2181, 3306, 6379, 8080, 9090, 3000, 9092, 29092 未被占用
4. 安全: 生产环境需要设置更强的密码和安全配置
5. 监控: 添加了 cAdvisor 和 Node Exporter 用于监控容器和主机资源
6. 扩展: 可轻松扩展 Kafka 集群，修改 docker-compose.yml 添加更多副本

这个方案既提供了逐步部署的验证方法，也提供了完整的 Docker Compose 统一部署方案，适合学习和生产使用。
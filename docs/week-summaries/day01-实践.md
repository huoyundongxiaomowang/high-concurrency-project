# 创建网络环境 
* docker network create app-network

## 启动 ZooKeeper
docker run -d --name zookeeper --network app-network -p 2181:2181 -e ALLOW_ANONYMOUS_LOGIN=yes -v zookeeper_data:/home/zookeeper zookeeper:latest

## 启动kafka
docker run -d \
--name kafka-kraft \
--network app-network \
-p 9092:9092 \
-p 9093:9093 \
-e KAFKA_NODE_ID=1 \
-e KAFKA_PROCESS_ROLES=broker,controller \
-e KAFKA_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093 \
-e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka-kraft:9092 \
-e KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER \
-e KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT \
-e KAFKA_CONTROLLER_QUORUM_VOTERS=1@kafka-kraft:9093 \
-e CLUSTER_ID=1 \
-e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
-e KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=1 \
apache/kafka:latest

### 创建主题
# 2. 进入容器测试生产者/消费者
docker exec -it kafka-kraft bash
#### 创建主题
在apache/kafka中 命令行在 /opt/kafka/bin 目录下，所有要加前缀

/opt/kafka/bin/kafka-topics.sh --create --topic test-topic --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
#### 列出主题
/opt/kafka/bin/kafka-topics.sh --list --bootstrap-server localhost:9092
#### 退出容器
exit

# 启动 Redis
docker run -d \
--name redis \
--network app-network \
-p 6379:6379 \
-e REDIS_PASSWORD=123456 \
-v redis_data:/home/redis \
redis:7-alpine redis-server --requirepass 123456 --appendonly yes

# 验证 Redis
docker exec -it redis redis-cli -a yourpassword ping
# 应返回: PONG

# 启动 MySQL
docker run -d \
--name mysql \
--network app-network \
-p 3306:3306 \
-e MYSQL_ROOT_PASSWORD=123456 \
-e MYSQL_DATABASE=appdb \
-e MYSQL_USER=appuser \
-e MYSQL_PASSWORD=123456 \
-v mysql_data:/var/lib/mysql \
mysql:8

# 验证 MySQL
docker exec -it mysql mysql -uroot -p123456 -e "SHOW DATABASES;"

# 启动 Kafka-UI (使用 provectuslabs/kafka-ui)
docker run -d \
--name kafka-ui \
--network app-network \
-p 8080:8080 \
-e KAFKA_CLUSTERS_0_NAME=local-kafka \
-e KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka-kraft:9092 \
provectuslabs/kafka-ui:latest

# 验证: 访问 http://localhost:8080


# 创建 prometheus.yml

sudo tee /etc/prometheus/prometheus.yml > /dev/null << 'EOF'
global:
scrape_interval: 15s
evaluation_interval: 15s

scrape_configs:
- job_name: 'prometheus'
  static_configs:
    - targets: ['localhost:9090']

- job_name: 'kafka'
  static_configs:
    - targets: ['kafka-kraft:9092']

- job_name: 'node-exporter'
  static_configs:
    - targets: ['node-exporter:9100']

- job_name: 'cadvisor'
  static_configs:
    - targets: ['cadvisor:8080']
EOF


# 启动 Prometheus
docker run -d \
--name prometheus \
--network app-network \
-p 9090:9090 \
-v /etc/prometheus:/etc/prometheus \
-v prometheus_data:/home/prometheus \
prom/prometheus:latest


# 启动 Grafana
docker run -d \
--name grafana \
--network app-network \
-p 3000:3000 \
-e GF_SECURITY_ADMIN_PASSWORD=admin \
-v grafana_data:/var/lib/grafana \
grafana/grafana-enterprise:latest
运行 docker-compose.yaml 文件非常简单，只需几个步骤。以下是详细的操作指南：

前提条件

确保您的系统已安装：
1. Docker - 版本 20.10.0 或更高
2. Docker Compose - 版本 1.29.0 或更高

检查安装：
docker --version
docker-compose --version


运行步骤

1. 准备文件

创建一个项目目录，将所有配置文件放在一起：
mkdir my-docker-app
cd my-docker-app


将您的 docker-compose.yaml 文件保存到该目录。

2. 创建 Prometheus 配置文件

在同一目录下创建 prometheus.yml 文件：
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
    - targets: ['kafka-kraft:9092']

- job_name: 'node-exporter'
  static_configs:
    - targets: ['node-exporter:9100']

- job_name: 'cadvisor'
  static_configs:
    - targets: ['cadvisor:8080']
      EOF


3. 启动所有服务

在项目目录中执行：
docker-compose up -d


参数说明：
• -d：后台运行（detached mode）

• 不加 -d：前台运行，可以看到所有日志输出

4. 查看运行状态

# 查看所有容器状态
docker-compose ps

# 查看日志（可指定服务名）
docker-compose logs
docker-compose logs kafka
docker-compose logs -f  # 实时跟踪日志

# 查看容器资源使用情况
docker-compose top


5. 停止服务

# 停止但不删除容器
docker-compose stop

# 停止并删除容器（保留数据卷）
docker-compose down

# 停止并删除容器、网络、数据卷（数据会丢失！）
docker-compose down -v


常用命令速查

命令 功能

docker-compose up -d 启动所有服务

docker-compose down 停止并删除所有服务

docker-compose ps 查看服务状态

docker-compose logs [服务名] 查看日志

docker-compose exec [服务名] [命令] 在容器内执行命令

docker-compose restart [服务名] 重启指定服务

docker-compose build 重新构建镜像（如有Dockerfile）

docker-compose pull 拉取最新镜像

验证服务是否正常运行

1. Kafka UI：访问 http://localhost:8080
2. Grafana：访问 http://localhost:3000
   • 用户名：admin

   • 密码：admin

3. Prometheus：访问 http://localhost:9090
4. MySQL：连接 localhost:3306
5. Redis：连接 localhost:6379

注意事项

1. 端口冲突：如果端口已被占用，修改 docker-compose.yaml 中的端口映射，如 "8081:8080"。
2. 数据持久化：使用 docker-compose down 不会删除数据卷，数据会保留。
3. 环境变量：敏感信息（如密码）建议使用环境变量文件：
   # 创建 .env 文件
   echo "MYSQL_ROOT_PASSWORD=your_password" > .env

   然后在 docker-compose.yaml 中引用：${MYSQL_ROOT_PASSWORD}

4. 网络问题：如果容器间无法通信，检查网络配置：
   docker network ls
   docker network inspect my-docker-app_app-network


快速测试

最简单的测试方法是创建一个测试文件：
# 创建测试目录和文件
mkdir test-docker-compose
cd test-docker-compose

# 创建 docker-compose.yaml（使用简化版本测试）
cat > docker-compose.yaml << 'EOF'
version: "3.8"
services:
redis:
image: redis:alpine
ports:
- "6379:6379"
EOF

# 启动测试
docker-compose up -d


这样您就可以快速熟悉 Docker Compose 的基本操作了。
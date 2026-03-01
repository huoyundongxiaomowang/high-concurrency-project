docker-compose 中有哪些服务？为什么需要它？
1. docker-compose 中有kafka,redis,mysql,kafka-ui,promotheus,grafana,
其中不需要zookeeper,kafka采用kraft模式，就可以不需要注册中心

在高并发的项目中，kafka可以削峰，解耦，
redis 可以降低直接查询mysql的查询数，同时提高访问速度，promotheus和grafana可以详细监控整个事件驱动中
各中间件的情况

每个服务暴露了哪些端口？为什么？

redis 6379,
kafka 9092
mysql 3306
grafana 3000
prometheus 9090
kafka-ui 8080
都采用各服务的默认端口

Kafka 在当前阶段承担什么责任？

kafka 主要用来削峰

启动过程中遇到的第一个问题是什么？怎么解决的？

问题是kafka-ui 启动时depends_on kafka-kraft 找不到服务，
依赖的是service 而不是container_name

如果这个 compose 出问题，你第一反应会查哪里？

我会看日志报错，里面会有详情